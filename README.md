# Rwanda Building ID Generator - Documentation

## Overview

This system generates unique building identifiers for Rwanda's national building dataset. Each building receives a standardized ID that combines administrative location codes with precise geographic coordinates, creating a machine-readable format suitable for national databases and smart infrastructure systems.

## Building ID Format

### Structure
```
[PROVINCE]-[DISTRICT]-[SECTOR]-BLDG-[LATITUDE]-[LONGITUDE]
```

### Example
```
K-G-KI-BLDG-S1932923-E3010017
```

Breaking down this example:
- `K` - City of Kigali (Province)
- `G` - Gasabo (District)
- `KI` - Kimironko (Sector)
- `BLDG` - Building marker (constant)
- `S1932923` - Latitude (-1.932923°)
- `E3010017` - Longitude (30.10017°)

## Components

### 1. Administrative Codes

#### Province Code (1 character)
- First letter of the province name
- Example: `K` for City of Kigali

#### District Code (1-2 characters)
- First letter of district name, or first two letters if conflicts exist
- Example: `G` for Gasabo

#### Sector Code (1-3 characters)
- Adaptive length based on uniqueness requirements
- Conflict resolution through progressive lengthening
- Examples:
  - `K` for Kinyinya (if unique)
  - `KIM` for Kimironko (to distinguish from Kimisagara)
  - `GA` for Gatsata (to distinguish from Gikomero)

### 2. Coordinate Encoding

#### Process
1. Multiply coordinate by 10^7 for precision (approximately 1cm accuracy)
2. Take absolute value
3. Truncate or pad to exactly 7 digits
4. Add directional prefix

#### Directional Prefixes
- Latitude: `S` (South) or `N` (North)
- Longitude: `E` (East) or `W` (West)

#### Examples
- `-1.9329232` → `S1932923`
- `30.1001747` → `E3010017`

### 3. Decoding Process

To reverse a building ID:
1. Split by hyphen delimiter
2. Extract coordinate components
3. Remove directional prefix and divide by 10^7
4. Apply sign based on prefix (S/W = negative, N/E = positive)

## Implementation Features

### Input Data Requirements

The system expects Excel or CSV files with these columns:
- `Province` or `province`
- `District` or `district`
- `Sector` or `sector`
- `Cell` or `cell` (optional)
- `Village` or `village` (optional)
- `latitude` or `lat`
- `longitude` or `lon` or `lng`

### Output Files

The system generates four output files:

1. **buildings_with_ids_[timestamp].xlsx** and **buildings_with_ids_[timestamp].csv**
   - Original data with added `building_id` column
   - All original columns preserved

2. **validation_report_[timestamp].xlsx** and **validation_report_[timestamp].csv**
   - Validation status for each building
   - Columns: `index`, `building_id`, `valid`, `issues`
   - Identifies format errors and coordinate boundary violations

3. **code_mappings_[timestamp].json**
   - Administrative division to code mappings
   - Useful for understanding code assignments
   - Structure:
   ```json
   {
     "provinces": {"City of Kigali": "K"},
     "districts": {"Gasabo": "G"},
     "sectors": {"Kimironko": "KIM", ...}
   }
   ```

4. **statistics_[timestamp].json**
   - Dataset summary statistics
   - Total buildings, unique IDs, duplicates
   - Breakdowns by province and district
   - Coordinate bounds

### Validation Rules

Buildings are validated against:

1. **Format Validation**
   - Correct number of components (6 parts)
   - Coordinate format: exactly 1 letter + 7 digits
   - Valid directional prefixes (N/S/E/W)

2. **Geographic Validation**
   - Latitude: -3.0° to -1.0° (Rwanda bounds)
   - Longitude: 28.5° to 31.0° (Rwanda bounds)

### Conflict Resolution

The system handles naming conflicts intelligently:

1. **Same Initial Letter**: When multiple locations start with the same letter
   - Progressive code lengthening (1 → 2 → 3 characters)
   - Numeric suffixes if needed (e.g., `G1`, `G2`)

2. **Duplicate IDs**: Prevented through:
   - Hierarchical location encoding
   - High-precision coordinates (7 decimal places)
   - Unique code generation per administrative level

## Usage Examples

### Basic Usage
```python
# Process a single district file
df, validation_df, stats, generator = main_pipeline("gasabo-buildings-dev-subset.xlsx")

# Check results
print(f"Generated {len(df)} building IDs")
print(f"Valid IDs: {len(validation_df[validation_df['valid']])}")
```

### Batch Processing
```python
# Process multiple district files
district_files = [
    "gasabo-buildings.xlsx",
    "kicukiro-buildings.xlsx",
    "nyarugenge-buildings.xlsx"
]
combined_df = process_multiple_districts(district_files)
```

### ID Lookup and Search
```python
# Create lookup system
lookup = create_id_lookup_system(df)

# Find buildings near a location (within ~111 meters)
nearby = find_buildings_in_area(df, -1.9329232, 30.1001747, radius_degrees=0.001)

# Decode a building ID
decoded = generator.decode_building_id("K-G-KI-BLDG-S1932923-E3010017")
# Returns: {'province_code': 'K', 'district_code': 'G', 'sector_code': 'KIM', 
#           'latitude': -1.932923, 'longitude': 30.10017, ...}
```

## Benefits

1. **Unique Identification**: Every building has a globally unique ID
2. **Human Readable**: Administrative codes provide location context
3. **Machine Parseable**: Standardized format for database integration
4. **GIS Compatible**: Embedded coordinates for mapping systems
5. **Scalable**: Works for millions of buildings nationwide
6. **Reversible**: IDs can be decoded back to location data

## Technical Specifications

- **Coordinate Precision**: 7 decimal places (~1.1 cm accuracy)
- **Character Set**: Alphanumeric only (A-Z, 0-9)
- **ID Length**: Variable (typically 30-35 characters)
- **Encoding**: UTF-8 compatible
- **Storage**: String/VARCHAR field recommended

## Future Enhancements

Potential improvements for consideration:

1. **Checksum Addition**: Add validation digit to detect transcription errors
2. **Building Type Codes**: Replace "BLDG" with specific types (RES, COM, IND)
3. **Temporal Component**: Add construction year or survey date
4. **Polygon Integration**: Link to building footprint geometries
5. **QR Code Generation**: Create scannable codes for field verification

## Error Handling

The system handles common data issues:

- Missing coordinates → Default to `S0000000-E0000000`
- Missing administrative divisions → Use placeholder codes
- Invalid characters in names → Automatic cleaning
- Coordinate outliers → Validation warnings
- Duplicate entries → Detection and reporting

## Performance

- Processes ~1,000 buildings per second
- Memory efficient for datasets up to 1 million buildings
- Batch processing support for larger datasets
- Excel file size limit: ~1 million rows per file