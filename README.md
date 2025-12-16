Post-process G-code from OrcaSlicer to tag each tree support as a separate object for Klipper's exclude object feature.

This script scans the G-code for support structures and wraps each individual support tree's G-code 
in unique object start/end blocks. It also labels the main model as a separate object. This allows 
Klipper to cancel a failing support tree mid-print without affecting other supports or the model.

Features:
- Detects each support tree by analyzing travel and retraction patterns and layer-by-layer continuity.
- Wraps all G-code for each support tree in `EXCLUDE_OBJECT_START`/`EXCLUDE_OBJECT_END` blocks with unique names (Support_1, Support_2, ...).
- Ensures that canceling a support object stops all future toolpath for that support (across all layers).
- Leaves travel moves and global retractions outside object blocks so printer can still reposition and prime for remaining objects.
- Labels the model's G-code as "Model" object. Other extrusions like skirt or brim are left unmodified.
- Inserts logging comments in the output G-code (listing objects and indicating when each object starts) to help verify correctness.

Usage:
    python3 support_object_separator.py input.gcode [output.gcode]

If output file is not specified, the script will overwrite the input file with the processed G-code.






```
#!/usr/bin/env python3

import sys
import math

def process_gcode(lines):
    """Process G-code lines and return a list of output lines with object labels added."""
    # Data structures for tracking support clusters
    clusters = {}  # support cluster id -> {'id': int, 'last_x': float, 'last_y': float, 'last_layer': int}
    line_object = {}  # map line index -> object name or id ('model' or support id)
    next_support_id = 1

    # State variables
    current_layer = -1
    current_category = None   # 'support', 'model', or 'other'
    current_cluster_id = None  # support cluster currently being printed (within a contiguous segment)
    extruder_abs = True       # assume absolute E unless M83 is found
    last_e_pos = 0.0          # last extruder position (for absolute mode)
    
    # Parameters
    travel_threshold = 5.0    # distance threshold to link support segments across layers (in mm)
    
    # Pass 1: Identify support clusters and map each print move to an object
    for i, line in enumerate(lines):
        line_stripped = line.strip()
        # Track layer changes
        if line_stripped.startswith(';LAYER:'):
            # New layer encountered
            try:
                current_layer = int(line_stripped.split(':',1)[1])
            except:
                # If layer number is not directly parseable, increment
                current_layer += 1
            # Reset current support cluster for new layer (each layer printing starts fresh)
            current_cluster_id = None
            continue
        # Track type of extrusion (support vs model vs others)
        if line_stripped.startswith(';TYPE:'):
            type_info = line_stripped.split(':',1)[1].lower()
            if 'support' in type_info:
                current_category = 'support'
            elif any(word in type_info for word in ['wall', 'skin', 'infill', 'perimeter', 'solid', 'bridge', 'top', 'bottom']):
                current_category = 'model'
            else:
                current_category = 'other'
            continue
        # Track extruder mode commands
        if 'M82' in line_stripped:   # set extruder to absolute mode
            extruder_abs = True
            last_e_pos = 0.0
        if 'M83' in line_stripped:   # set extruder to relative mode
            extruder_abs = False
            last_e_pos = 0.0
        if line_stripped.startswith('G92'):
            # Reset extruder position (e.g., G92 E0)
            if 'E' in line_stripped:
                try:
                    e_val = float(line_stripped.split('E',1)[1].split()[0])
                except:
                    e_val = 0.0
                if abs(e_val) < 1e-9:
                    last_e_pos = 0.0

        # Only process print moves (skip non-print categories or preamble)
        if current_category not in ('support', 'model'):
            continue

        # Parse motion commands
        if line_stripped.startswith('G0') or line_stripped.startswith('G1'):
            # Initialize motion analysis flags
            move_extruding = False
            move_retracting = False
            # Track if XY move present
            x_move = None
            y_move = None
            # Extract coordinates and extrusion value
            e_value = None
            tokens = line_stripped.split()
            for token in tokens[1:]:
                if token.startswith('X'):
                    try:
                        x_move = float(token[1:])
                        # Update current position X (not strictly needed for logic beyond clustering)
                    except:
                        pass
                elif token.startswith('Y'):
                    try:
                        y_move = float(token[1:])
                        # Update current position Y
                    except:
                        pass
                elif token.startswith('E'):
                    try:
                        e_value = float(token[1:])
                    except:
                        e_value = None

            # Compute extrusion amount
            if e_value is not None:
                if extruder_abs:
                    # Absolute mode: compare to last known E position
                    extrude_amount = e_value - last_e_pos
                    last_e_pos = e_value
                else:
                    # Relative mode: the value is the extrusion amount directly
                    extrude_amount = e_value
                    last_e_pos += e_value
            else:
                extrude_amount = 0.0

            # Determine move type from extrusion amount
            if extrude_amount > 1e-6:
                move_extruding = True   # positive extrusion -> printing
            elif extrude_amount < -1e-6:
                move_retracting = True  # negative extrusion -> retract

            # Auto-detect extruder mode if needed:
            if extruder_abs and move_retracting and x_move is None and y_move is None and e_value is not None:
                # A pure retract move in what appears to be absolute mode likely indicates the slicer is using relative E without an explicit M83
                extruder_abs = False
                last_e_pos = 0.0  # reset extruder baseline for relative mode
                # Recompute extrude_amount in relative terms
                extrude_amount = e_value
                move_extruding = extrude_amount > 1e-6
                move_retracting = extrude_amount < -1e-6

            # Handle support category moves
            if current_category == 'support':
                if move_extruding:
                    if x_move is None and y_move is None:
                        # Extrusion with no XY motion: treat as unretract or prime, not actual support structure printing
                        move_extruding = False
                    else:
                        # Start of a new support segment if none current
                        if current_cluster_id is None:
                            # Attempt to match this support segment to an existing support tree from previous layer
                            assigned_id = None
                            for cid, info in clusters.items():
                                # Only consider clusters that were active recently (last seen in a recent layer)
                                if current_layer - info['last_layer'] > 2:
                                    continue
                                # Compute distance from this move to last position of cluster
                                dx = info['last_x'] - (x_move or info['last_x'])
                                dy = info['last_y'] - (y_move or info['last_y'])
                                if math.hypot(dx, dy) < travel_threshold:
                                    assigned_id = cid
                                    break
                            if assigned_id is None:
                                # No nearby existing cluster, start a new support tree
                                assigned_id = next_support_id
                                clusters[assigned_id] = {'id': assigned_id, 'last_x': 0.0, 'last_y': 0.0, 'last_layer': 0}
                                next_support_id += 1
                            current_cluster_id = assigned_id
                        # Mark this line as belonging to the current support cluster
                        line_object[i] = current_cluster_id
                        # Update cluster's last seen position
                        if x_move is not None and y_move is not None:
                            clusters[current_cluster_id]['last_x'] = x_move
                            clusters[current_cluster_id]['last_y'] = y_move
                            clusters[current_cluster_id]['last_layer'] = current_layer
                if move_retracting or (not move_extruding and e_value is not None):
                    # Any retraction or stand-alone extruder move indicates end of a support segment
                    current_cluster_id = None

            # Handle model category moves
            elif current_category == 'model':
                if move_extruding:
                    # Mark model extrusion lines as part of 'model' object
                    line_object[i] = 'model'
                if move_retracting:
                    # Retraction in model - just treat as end of a contiguous model segment (not needed to explicitly track separate clusters for model)
                    # (Model object will remain the same throughout, so no special cluster handling needed)
                    pass

    # Pass 2: Generate output G-code with object definitions and object blocks
    output_lines = []
    # Copy header/preamble lines (before first layer appears) directly
    first_layer_idx = 0
    for idx, line in enumerate(lines):
        if line.strip().startswith(';LAYER:'):
            first_layer_idx = idx
            break
        output_lines.append(line.rstrip('\n'))

    # Compute object definitions (centers for UI display)
    support_ids = sorted(cid for cid in clusters.keys())
    # Calculate cluster centers (using last seen coordinates or average if needed)
    cluster_centers = {}
    for cid in support_ids:
        cx = clusters[cid]['last_x']
        cy = clusters[cid]['last_y']
        cluster_centers[cid] = (cx, cy)
    # Calculate model object center (average of all model extrusion coordinates)
    model_center = None
    if any(obj == 'model' for obj in line_object.values()):
        sum_x = 0.0; sum_y = 0.0; count = 0
        for idx, obj in line_object.items():
            if obj == 'model':
                # Extract XY position from that line
                tokens = lines[idx].strip().split()
                x_val = None; y_val = None
                for token in tokens:
                    if token.startswith('X'):
                        try:
                            x_val = float(token[1:])
                        except:
                            pass
                    elif token.startswith('Y'):
                        try:
                            y_val = float(token[1:])
                        except:
                            pass
                if x_val is not None and y_val is not None:
                    sum_x += x_val; sum_y += y_val; count += 1
        if count > 0:
            model_center = (sum_x / count, sum_y / count)

    # Output EXCLUDE_OBJECT_DEFINE commands for each object at the top of file
    for cid in support_ids:
        cx, cy = cluster_centers[cid]
        output_lines.append(f"EXCLUDE_OBJECT_DEFINE NAME=Support_{cid} CENTER={cx:.2f},{cy:.2f}")
    if model_center is not None:
        # Use "Model" as object name for the printed model
        cx, cy = model_center
        output_lines.append(f"EXCLUDE_OBJECT_DEFINE NAME=Model CENTER={cx:.2f},{cy:.2f}")
    # Add summary comment listing objects
    if support_ids:
        support_list_str = ", ".join(f"Support_{cid}" for cid in support_ids)
        output_lines.append(f"; Support objects: {support_list_str}")
    if model_center is not None:
        output_lines.append(f"; Model object: Model")

    # Iterate through printing lines (from first layer onward) and insert object start/stop markers
    current_obj_name = None
    for idx in range(first_layer_idx, len(lines)):
        line = lines[idx].rstrip('\n')
        stripped = line.strip()
        if stripped.startswith(';LAYER:'):
            # Close any open object at layer change (one object should not span layer breaks in output)
            if current_obj_name is not None:
                output_lines.append(f"EXCLUDE_OBJECT_END NAME={current_obj_name}")
                current_obj_name = None
            output_lines.append(line)
            continue
        if stripped.startswith(';TYPE:'):
            # Pass through type comments (they are just informative)
            output_lines.append(line)
            continue

        # Determine if this line is part of a model or support object
        obj = line_object.get(idx)
        if obj is None:
            # No object mapping (likely travel, retract, or other moves)
            # Close any current object before non-object moves (to keep travels outside objects)
            if current_obj_name is not None:
                output_lines.append(f"EXCLUDE_OBJECT_END NAME={current_obj_name}")
                current_obj_name = None
            output_lines.append(line)
        else:
            # Determine object name
            obj_name = "Model" if obj == 'model' else f"Support_{obj}"
            # If starting a new object segment
            if current_obj_name is None:
                current_obj_name = obj_name
                output_lines.append(f"EXCLUDE_OBJECT_START NAME={obj_name}")
                output_lines.append(f"; Starting object {obj_name}")
            elif current_obj_name != obj_name:
                # Switching to a different object mid-layer
                output_lines.append(f"EXCLUDE_OBJECT_END NAME={current_obj_name}")
                current_obj_name = obj_name
                output_lines.append(f"EXCLUDE_OBJECT_START NAME={obj_name}")
                output_lines.append(f"; Starting object {obj_name}")
            output_lines.append(line)
    # Close any object that remains open at end of file
    if current_obj_name is not None:
        output_lines.append(f"EXCLUDE_OBJECT_END NAME={current_obj_name}")
        current_obj_name = None

    return output_lines

def main():
    if len(sys.argv) < 2:
        sys.stderr.write("Usage: {} <input_gcode> [output_gcode]\n".format(sys.argv[0]))
        sys.exit(1)
    input_path = sys.argv[1]
    output_path = sys.argv[2] if len(sys.argv) > 2 else input_path
    # Read input G-code
    try:
        with open(input_path, 'r') as f:
            lines = f.readlines()
    except Exception as e:
        sys.stderr.write(f"Error: Could not read file '{input_path}' ({e})\n")
        sys.exit(1)
    # Process the G-code lines
    output_lines = process_gcode(lines)
    # Write output G-code
    try:
        with open(output_path, 'w') as f:
            for line in output_lines:
                f.write(line + "\n")
    except Exception as e:
        sys.stderr.write(f"Error: Could not write output file '{output_path}' ({e})\n")
        sys.exit(1)

if __name__ == "__main__":
    main()

```
