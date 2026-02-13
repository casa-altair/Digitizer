# Digitizer Conversion Methods Documentation

## Overview

This document contains all available digitizer conversion methods for the Weighbridge Automation System. Each method is provided as a complete, standalone code snippet that can be copy-pasted into `conversion.py` at **line 570** (replacing the existing conversion logic).

## How to Use

1. Identify which format your digitizer uses (check logs for raw data format)
2. Find the corresponding method in this document
3. Copy the entire code block for that method
4. Open `Weighbridge-App/conversion.py`
5. **Delete all conversion methods** (from line 570 to approximately line 945)
6. **Paste the selected method** at line 570
7. Save and test

**Important:** Only use ONE method at a time. Having multiple methods can cause incorrect matches.

---

## Method 1: Bracketed Format (Highest Priority)

**Format:** `[000000][000000][000000]...` (multiple weight readings in brackets, single line)

**Examples:**
- `[000000][000000][000000][000000][000000]` → 0 kg
- `[000080][000080][000080][000080][000080]` → 0.08 kg (80g)
- `[001500][001500][001500]` → 1.5 kg (1500g)

**Code:**
```python
            # Method: Bracketed format
            # Format: [000000][000000][000000]... (multiple weight readings in brackets, single line)
            # Extract the last (most recent) bracketed value
            if weight is None:
                # Find all bracketed values: [digits]
                bracket_matches = re.findall(r'\[(\d+)\]', text)
                if bracket_matches:
                    # Use the last (most recent) bracketed value
                    weight_str = bracket_matches[-1]
                    weight = float(weight_str) / 1000.0  # Convert grams to kg (000080 = 80g = 0.08kg)
                    logger.info(f"Weight decoded via bracketed format: {weight} kg from '[{weight_str}]' (found {len(bracket_matches)} brackets, using last)")
```

---

## Method 2: "kg" Format

**Format:** `{spaces}{number} kg    G 000000 A`

**Examples:**
- `     20 kg    G 000000 A` → 20 kg
- `     00 kg    G 000000 A` → 0 kg
- `    -40 kg    G 000000 A` → -40 kg

**Code:**
```python
            # Method: "kg" format
            # Format: {spaces}{number} kg    G 000000 A
            # Pattern: optional spaces, optional minus sign, digits, space, "kg"
            if weight is None:
                match = re.search(r'([+-]?\d+)\s+kg', text)
                if match:
                    weight = float(match.group(1))
                    logger.debug(f"Weight decoded via 'kg' format: {weight} kg from '{match.group(0)}'")
```

---

## Method 3: "Wt:" Format

**Format:** `Wt:     {spaces}{weight_digits}{non-digit}Wt:...`

**Examples:**
- `Wt:  58940♥0⚠Wt:` → 58940 kg (stops at ♥)
- `Wt:     50G0GWt:` → 50 kg (stops at G)

**Code:**
```python
            # Method: "Wt:" format
            # Format: Wt:     {spaces}{weight_digits}{non-digit}Wt:...
            # Logic:
            #   1. Accumulate data in buffer until we find "Wt:"
            #   2. After "Wt:" (after the colon), skip spaces
            #   3. Extract all consecutive digits until first non-digit
            #   4. Return the weight as-is (no auto-division)
            
            # Add current data to buffer
            self._raw_buffer += raw_data
            
            # Limit buffer size to prevent memory issues (keep last 200 bytes)
            if len(self._raw_buffer) > 200:
                self._raw_buffer = self._raw_buffer[-200:]
            
            # Find the LAST "Wt:" in the buffer (most recent reading)
            wt_pattern = b'Wt:'
            wt_pos = -1
            search_pos = 0
            # Find the last occurrence of "Wt:"
            while True:
                pos = self._raw_buffer.find(wt_pattern, search_pos)
                if pos == -1:
                    break
                wt_pos = pos
                search_pos = pos + 1
            
            # If not found, try lowercase
            if wt_pos == -1:
                wt_pattern = b'wt:'
                search_pos = 0
                while True:
                    pos = self._raw_buffer.find(wt_pattern, search_pos)
                    if pos == -1:
                        break
                    wt_pos = pos
                    search_pos = pos + 1
            
            if wt_pos >= 0:
                # Found "Wt:", now read after the colon
                after_colon_pos = wt_pos + 3
                
                if after_colon_pos < len(self._raw_buffer):
                    # Extract portion after "Wt:"
                    after_wt = self._raw_buffer[after_colon_pos:]
                    
                    # Decode to string
                    after_str = after_wt.decode('ascii', errors='ignore')
                    
                    # Skip ONLY spaces at the start
                    start_idx = 0
                    while start_idx < len(after_str) and after_str[start_idx] == ' ':
                        start_idx += 1
                    
                    # Extract all consecutive digits from this position
                    weight_digits = ""
                    found_separator = False
                    for i in range(start_idx, len(after_str)):
                        char = after_str[i]
                        if char.isdigit():
                            weight_digits += char
                        else:
                            # Stop at first non-digit character
                            found_separator = True
                            break
                    
                    # Only extract if we have digits AND we found a non-digit separator
                    if weight_digits and found_separator:
                        weight = float(weight_digits)
                        logger.debug(f"Weight decoded via Wt: format: {weight} kg from '{weight_digits}'")
                        
                        # Clear buffer after successful extraction
                        if len(self._raw_buffer) > 30:
                            self._raw_buffer = self._raw_buffer[-30:]
                        else:
                            self._raw_buffer = b""
                    elif weight_digits and not found_separator:
                        # Check if there's another "Wt:" pattern after this one
                        next_wt_pos = self._raw_buffer.find(wt_pattern, wt_pos + 1)
                        if next_wt_pos >= 0:
                            # There's another "Wt:" after this one, so this reading is complete
                            weight = float(weight_digits)
                            logger.debug(f"Weight decoded via Wt: format: {weight} kg from '{weight_digits}' (no separator but next Wt: found)")
                            
                            # Clear buffer after successful extraction
                            if len(self._raw_buffer) > 30:
                                self._raw_buffer = self._raw_buffer[-30:]
                            else:
                                self._raw_buffer = b""
```

---

## Method 4: STX/ETX Framed Data (STX at beginning)

**Format:** `STX STX + spaces + digits` (e.g., `\x02\x02 0049510`)

**Examples:**
- `\x02\x02 0049510` → 49510 kg
- `\x02\x02\x02 0002000` → 2000 kg

**Code:**
```python
            # Method: STX/ETX framed data (STX at beginning)
            # Format: STX STX + spaces + digits (e.g., "\x02\x02 0049510")
            if weight is None and (b'\x02' in raw_data or b'\x02' in self._raw_buffer):
                # Check the buffer (accumulated data) for complete patterns
                buffer_to_check = self._raw_buffer if len(self._raw_buffer) > len(raw_data) else raw_data
                
                # Check for STX at the beginning
                if buffer_to_check.startswith(b'\x02'):
                    # Count consecutive STX characters
                    stx_count = 0
                    pos = 0
                    while pos < len(buffer_to_check) and buffer_to_check[pos] == 0x02:
                        stx_count += 1
                        pos += 1
                    
                    # Skip spaces after STX
                    while pos < len(buffer_to_check) and buffer_to_check[pos] in (0x20, 0x00):
                        pos += 1
                    
                    # Extract digits from this position
                    if pos < len(buffer_to_check):
                        remaining = buffer_to_check[pos:]
                        try:
                            remaining_str = remaining.decode('ascii', errors='ignore')
                            # Match digits (with optional leading zeros)
                            digit_match = re.match(r'([+-]?\d+)', remaining_str)
                            if digit_match:
                                weight_str = digit_match.group(1)
                                weight = float(weight_str)
                                logger.debug(f"Weight decoded via STX+spaces+digits format: {weight} kg from '{weight_str}' (STX count: {stx_count})")
                                # Clear buffer after successful extraction
                                if len(self._raw_buffer) > 30:
                                    self._raw_buffer = self._raw_buffer[-30:]
                                else:
                                    self._raw_buffer = b""
                        except:
                            pass
```

---

## Method 5: STX/ETX Framed Data (STX at end)

**Format:** `digits + STX STX` (e.g., `0000050\x02\x02` or ` 0044050\x02\x02`)

**Examples:**
- `0000050\x02\x02` → 50 kg
- ` 0044050\x02\x02` → 44050 kg
- `0000000\x02\x02` → 0 kg

**Code:**
```python
            # Method: STX/ETX framed data (STX at end)
            # Format: digits + STX STX (e.g., "0000050\x02\x02" or " 0044050\x02\x02")
            if weight is None and (b'\x02' in raw_data or b'\x02' in self._raw_buffer):
                # Check the buffer (accumulated data) for complete patterns
                buffer_to_check = self._raw_buffer if len(self._raw_buffer) > len(raw_data) else raw_data
                
                # Check for STX at the end
                if buffer_to_check.endswith(b'\x02'):
                    # Find the last STX position
                    last_stx_pos = buffer_to_check.rfind(b'\x02')
                    if last_stx_pos > 0:
                        # Check if there are multiple STX at the end (need at least 2 STX)
                        stx_count = 0
                        pos = last_stx_pos
                        while pos >= 0 and buffer_to_check[pos] == 0x02:
                            stx_count += 1
                            pos -= 1
                        stx_start = pos + 1
                        
                        # Need at least 2 STX characters to consider this a valid pattern
                        if stx_count >= 2:
                            # Extract data before STX
                            data_before_stx = buffer_to_check[:stx_start]
                            
                            # Skip leading spaces (but keep leading zeros)
                            digit_start = 0
                            while digit_start < len(data_before_stx) and data_before_stx[digit_start] == 0x20:
                                digit_start += 1
                            
                            # Extract digits (including leading zeros)
                            if digit_start < len(data_before_stx):
                                remaining = data_before_stx[digit_start:]
                                try:
                                    remaining_str = remaining.decode('ascii', errors='ignore')
                                    # Match digits (with optional leading zeros)
                                    digit_match = re.match(r'([+-]?\d+)', remaining_str)
                                    if digit_match:
                                        weight_str = digit_match.group(1)
                                        weight = float(weight_str)
                                        logger.debug(f"Weight decoded via digits+STX format: {weight} kg from '{weight_str}' (STX count: {stx_count} at end)")
                                        # Clear buffer after successful extraction
                                        if len(self._raw_buffer) > 30:
                                            self._raw_buffer = self._raw_buffer[-30:]
                                        else:
                                            self._raw_buffer = b""
                                except:
                                    pass
```

---

## Method 6: STX/ETX Framed (Standard)

**Format:** `STX (0x02) + data + ETX (0x03)`

**Examples:**
- `\x02 12345 \x03` → 12345 kg
- `\x02 00000 \x03` → 0 kg

**Code:**
```python
            # Method: STX/ETX framed (standard)
            # Format: STX (0x02) + data + ETX (0x03)
            if weight is None and (b'\x02' in raw_data or b'\x03' in raw_data):
                # Remove STX/ETX
                text_clean = text.replace('\x02', '').replace('\x03', '')
                text_clean = text_clean.replace('\r', '').replace('\n', '').strip()
                
                # Check if it's all spaces/empty (weight = 0)
                if len(text_clean) == 0 or text_clean.strip() == '':
                    weight = 0.0
                    logger.debug("STX/ETX data is empty/spaces only, weight = 0")
                else:
                    # Extract numeric value
                    match = re.search(r'[-+]?\s*(\d+(?:\.\d+)?)', text_clean)
                    if match:
                        weight = float(match.group(1))
                        logger.debug(f"Weight decoded via STX/ETX method: {weight} kg")
```

---

## Method 7: Character-by-Character Accumulation

**Format:** One character at a time, accumulates until complete

**Examples:**
- Receives: `'1'`, `'2'`, `'3'`, `'4'`, `'5'`, `'\n'` → 12345 kg

**Code:**
```python
            # Method: Character-by-character accumulation
            # Some digitizers send one character at a time
            # We accumulate in buffer until we get a complete reading
            if weight is None and (len(raw_data) == 1 or len(raw_data) <= 3):
                # Accumulate characters
                self._buffer += text.replace('\r', '').replace('\n', '')
                
                # Check if we have a complete number (ends with newline or buffer is long enough)
                if '\n' in text or '\r' in text or len(self._buffer) >= 8:
                    # Try to extract weight from buffer
                    match = re.search(r'[-+]?\s*(\d+(?:\.\d+)?)', self._buffer)
                    if match:
                        weight = float(match.group(1))
                        logger.debug(f"Weight decoded via buffer method: {weight} kg")
                    # Clear buffer
                    self._buffer = ""
```

---

## Method 8: ST,GS Format

**Format:** `ST,GS,{sign}{digits}kg`

**Examples:**
- `ST,GS,+0040740kg` → 40740 kg
- `ST,GS,-0012345kg` → -12345 kg

**Code:**
```python
            # Method: ST,GS format
            # Format: ST,GS,{sign}{digits}kg
            if weight is None:
                text_clean = text.replace('\r', '').replace('\n', '').strip()
                # Check for ST,GS format
                if text_clean.startswith('ST,GS,') or 'ST,GS,' in text_clean:
                    # Extract the numeric part after ST,GS,
                    match = re.search(r'ST,GS,([+-]?\d+(?:\.\d+)?)', text_clean)
                    if match:
                        weight = float(match.group(1))
                        logger.debug(f"Weight decoded via ST,GS format: {weight} kg")
```

---

## Method 9: Weight Prefix Format

**Format:** `{weight}{control_chars}{trailing_chars}`

**Examples:**
- `0`PF%q▒▒` → 0 kg
- `50ePG%w▒▒` → 50 kg

**Code:**
```python
            # Method: Weight prefix format
            # Format: {weight}{control_chars}{trailing_chars}
            if weight is None:
                text_clean = text.replace('\r', '').replace('\n', '').strip()
                # Try to match weight at the beginning of the string
                match = re.match(r'^([+-]?\d+(?:\.\d+)?)', text_clean)
                if match:
                    weight = float(match.group(1))
                    logger.debug(f"Weight decoded via prefix format: {weight} kg (from '{text_clean[:20]}...')")
```

---

## Method 10: Plain ASCII with Spaces/Padding

**Format:** `{spaces}{digits}` or `{spaces}{digits}.{decimals} kg`

**Examples:**
- `  12345` → 12345 kg
- ` 12345.00 kg` → 12345 kg
- ` 0002000` → 2000 kg (leading zeros preserved)

**Code:**
```python
            # Method: Plain ASCII with spaces/padding
            # Format: "  12345" or " 12345.00 kg" or " 0002000" (space + digits)
            if weight is None:
                # Don't strip leading spaces yet - we need them for the pattern
                text_no_crlf = text.replace('\r', '').replace('\n', '')
                # Remove common units
                text_clean = re.sub(r'[kKgG]', '', text_no_crlf)
                
                # Match pattern: optional sign, optional spaces, digits
                match = re.search(r'[-+]?\s*(\d+(?:\.\d+)?)', text_clean)
                if match:
                    weight_str = match.group(1)
                    weight = float(weight_str)
                    logger.debug(f"Weight decoded via plain ASCII method: {weight} kg from '{weight_str}' (raw: '{text_clean[:30]}...')")
                else:
                    logger.warning(f"Plain ASCII method failed - text: '{text_clean[:50]}...'")
```

---

## Implementation Instructions

### Step 1: Identify Your Format

Check your application logs for the raw digitizer data. Look for lines like:
```
Digitizer raw data: '[000080][000080]...' (hex: ...)
```

### Step 2: Select the Method

Find the method in this document that matches your format.

### Step 3: Replace the Code

1. Open `Weighbridge-App/conversion.py`
2. Find line 570 (starts with `# Method 7: Bracketed format...`)
3. **Delete everything** from line 570 to approximately line 945 (before `# =================================================================`)
4. **Paste your selected method** at line 570
5. Ensure proper indentation (should be at the same level as the deleted code)

### Step 4: Test

Run the application and verify weight readings are correct.

---

## Notes

- **Only use ONE method** - having multiple methods can cause incorrect matches
- The `self._raw_buffer` and `self._buffer` variables are used by some methods for accumulation
- All methods should set `weight` variable to a float value (in kg)
- If conversion fails, `weight` should remain `None`
- Logging is included for debugging - adjust log levels as needed

---

## Troubleshooting

**Problem:** Wrong weight is being extracted
- **Solution:** Check logs to see which method matched. Try a more specific method or adjust the regex pattern.

**Problem:** Weight is always None
- **Solution:** Check the raw data format in logs. Your format might not match any method - you may need to create a custom method.

**Problem:** Weight is off by a factor (e.g., 10x or 100x)
- **Solution:** Some formats need division. For example, Method 1 (Bracketed) divides by 1000.0 to convert grams to kg. Adjust the conversion factor if needed.

---

## Version

Document Version: 1.0.0  
Last Updated: 2026-02-11
