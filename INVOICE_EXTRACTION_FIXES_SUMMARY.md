# Invoice Extraction Fixes - Complete Summary

## Overview
Fixed critical issues with PDF invoice extraction to properly handle:
- Address field cleanup (removing mixed-in labels)
- Phone number extraction and validation
- Reference field extraction
- Line item parsing with proper qty/rate/value assignment
- Monetary values (subtotal, tax, total)

## Changes Made

### 1. PDF Text Extractor (`tracker/utils/pdf_text_extractor.py`)

#### Address Extraction (Lines 433-463)
**Problem**: Address field was mixing in unwanted labels like "Cust Ref : DAR ES SALAAM Ref Date : TANZANIA Del. Date : 25/10/2025"

**Solution**:
- Added regex cleanup to remove trailing label indicators from address lines
- Checks for patterns like "Cust", "Ref", "Date", "Del" and removes everything after them
- Validates each continuation line for embedded labels before adding to address
- Final cleanup removes any remaining "Cust Ref", "Ref Date", "Del Date" patterns

**Example**:
```
Before: "P.O. BOX 15950 Cust Ref : DAR ES SALAAM Ref Date : TANZANIA Del. Date : 25/10/2025"
After:  "P.O. BOX 15950 DAR ES SALAAM TANZANIA"
```

#### Phone Extraction (Lines 465-481)
**Problem**: Phone field showing "-" instead of extracting actual phone number "0784 000 994"

**Solution**:
- Improved regex pattern to better match phone numbers
- Added digit validation (must have at least 7 digits)
- Validates phone numbers don't contain product codes/specifications
- Cleans up trailing non-phone characters

**Example**:
```
Before: "-"
After:  "0784 000 994"
```

#### Reference Extraction (Lines 483-512)
**Problem**: Reference field not being properly extracted for patterns like "FOR T 290 EFQ"

**Solution**:
- Added specific pattern for "FOR T" references (common in these invoices)
- Avoids confusing "Cust Ref" (customer reference in address) with "Reference"
- Properly filters out date-like values

**Example**:
```
Before: None
After:  "FOR T 290 EFQ"
```

#### Line Item Parsing (Lines 623-796)
**Problem**: Line items were mixing description with unit types and discount percentages
- Item 1: "BF GOODRICH LT265/65R17 116/113S TL ALL-TERRAIN T/A KO3 LRD RWL GO" - good
- Item 1 was showing percentage "18.00%" as part of description - bad
- Item 4: "WHEEL ALIGNMENT SMALL 50.00%" - bad, discount mixed into description

**Solution**:
- Separate percentages from quantity/rate/value numbers
- Improved unit detection to stop description at unit keywords
- Better handling of multi-line items that span across qty/unit lines
- More intelligent assignment of numbers to qty, rate, and value fields

**Example**:
```
Before: Item contains "18.00%" in description
After:  Item description clean, percentage stored separately

Item 1:
  Description: "BF GOODRICH TYRE LT265/65R17 116/113S TL ALL-TERRAIN T/A KO3 LRD RWL GO"
  Qty: 4
  Unit: PCS
  Rate: 1,037,400.00
  Value: 3,402,672.00
```

### 2. Invoice Creation Template (`tracker/templates/tracker/invoice_create.html`)

#### Added Line Items Display Section (New)
**Purpose**: Show extracted line items to user for review before invoice creation

**Features**:
- Table showing Code, Description, Unit, Qty, Unit Price, Line Total
- Hidden form fields for each line item that get submitted with the form
- Conditional display (shows only if items were extracted)

**Fields Submitted**:
- `item_code[]` - Item code
- `item_description[]` - Item description
- `item_unit[]` - Unit of measure (PCS, UNT, etc.)
- `item_qty[]` - Quantity
- `item_price[]` - Unit price (rate)

#### Added Monetary Values Handling (Updated)
**Purpose**: Ensure extracted subtotal, tax, and total are properly stored and submitted

**Changes**:
- Added hidden form fields for monetary values
- Fields: `subtotal`, `tax_amount`, `total_amount`
- Values are populated from extraction and submitted with the form
- Preview box shows extracted totals with proper formatting

#### Enhanced Extraction Preview (Updated)
**Purpose**: Show user what was extracted before proceeding

**Improvements**:
- Shows extracted monetary totals in preview
- Displays seller information (if available)
- Shows customer info and invoice details
- Shows all line items in a readable table format

### 3. API Endpoint (`tracker/views_invoice_upload.py`)

#### Monetary Value Mapping
The API correctly maps extraction results:
- `subtotal` → form field `subtotal`
- `tax` → form field `tax_amount`
- `total` → form field `total_amount`

All values are properly converted to Decimal for database storage.

## Testing

### Test Cases Covered

#### Test 1: PI - 1765632.pdf (STATEOIL TANZANIA LIMITED)
**Expected Results**:
- Code No: A01696 ✓
- Customer: STATEOIL TANZANIA LIMITED ✓
- Address: "P.O. BOX 15950 DAR ES SALAAM TANZANIA" (clean, no labels) ✓
- Phone: 0784 000 994 ✓
- Reference: FOR T 290 EFQ ✓
- Subtotal: 3,484,144.00 ✓
- Tax: 627,145.92 ✓
- Total: 4,111,289.92 ✓
- Line Items: 4 items ✓
  - Item 1: BF GOODRICH TYRE (without percentage in description) ✓
  - Item 2: VALVE (1214 TR 414) ✓
  - Item 3: WHEEL BALANCE ALLOYD ✓
  - Item 4: WHEEL ALIGNMENT SMALL (without percentage in description) ✓

#### Test 2: PI - 1764509.pdf (JTI LEAF SERVICES LIMITED)
**Expected Results**:
- Customer: JTI LEAF SERVICES LIMITED ✓
- Code No: A08054 ✓
- Phone: 0784 000 994 ✓
- Subtotal: 27,542.37 ✓
- Line Items: 2 items ✓

### Running Tests
```bash
python test_pdf_extraction_complete.py
```

## Data Flow

1. **User uploads PDF** → `api_extract_invoice_preview` endpoint
2. **PDF extracted** → `extract_from_bytes()` function in pdf_text_extractor.py
3. **Text parsed** → `parse_invoice_data()` extracts all fields
4. **JSON returned** → Frontend gets structured data
5. **Preview shown** → User sees extracted data in modal
6. **User confirms** → Data applied to form + line items populated
7. **Form submitted** → `api_create_invoice_from_upload` endpoint
8. **Invoice created** → With all extracted data and line items

## Form Field Names (Critical)

The following field names MUST be used for the backend to work correctly:

### Header Fields
- `customer_name` - Customer name
- `customer_phone` - Customer phone
- `customer_email` - Customer email  
- `customer_address` - Customer address
- `invoice_date` - Invoice date (YYYY-MM-DD format)
- `invoice_number` - Invoice/PI number
- `reference` - Reference field (e.g., "FOR T 290 EFQ")
- `attended_by` - Attended By
- `kind_attention` - Kind Attention

### Monetary Fields
- `subtotal` - Net value / subtotal
- `tax_amount` - VAT / Tax amount
- `total_amount` - Gross value / total

### Line Item Fields (Arrays)
- `item_code[]` - Item codes
- `item_description[]` - Item descriptions
- `item_qty[]` - Quantities
- `item_unit[]` - Units of measure
- `item_price[]` - Unit prices

### Seller Fields (Optional)
- `seller_name` - Seller/supplier name
- `seller_address` - Seller address
- `seller_phone` - Seller phone
- `seller_email` - Seller email
- `seller_tax_id` - Seller tax ID
- `seller_vat_reg` - Seller VAT registration

## Known Limitations

1. **OCR Not Available**: System uses PDF text extraction only, not OCR. PDFs must have selectable text.
2. **Format-Specific**: Extraction rules optimized for Superdoll Proforma Invoice format
3. **Manual Correction**: Users can edit all extracted data before creation
4. **Line Item Limits**: Form supports multiple line items but only first few are typically visible

## Future Improvements

1. Add OCR support for image-based PDFs
2. Support more invoice formats and layouts
3. Allow users to edit extracted line items in the table before submission
4. Add line item deletion/re-ordering functionality
5. Implement validation rules for monetary values
6. Add duplicate line item detection and merging

## Deployment Notes

1. No database migrations required
2. No new dependencies added
3. Changes are backward compatible
4. Existing invoices not affected
5. PDF extraction may require PyMuPDF or PyPDF2 library
