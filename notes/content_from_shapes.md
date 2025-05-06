Absolutely! Let's delve deeper into extracting content from various shapes—such as circles, ovals, rounded rectangles, and tilted rectangles—directly from your PDF using PyMuPDF, without relying on image conversion or OpenCV. This approach is particularly effective for AutoCAD-style PDFs, which often contain vector graphics and embedded text.

---

## 🧠 Understanding the Challenge

In PDFs, shapes like circles, ovals, and rectangles are typically drawn as vector graphics, not as text. PyMuPDF provides methods to extract these shapes and their associated content. However, detecting and extracting text within these shapes requires careful analysis of their geometric properties.

---

## 🔍 Step-by-Step Guide to Extracting Content from Shapes

### 1. **Extract Drawings from the PDF**

First, we'll extract the vector graphics (drawings) from the PDF. These drawings can include circles, ovals, rectangles, and other shapes.

```python
import fitz  # PyMuPDF

def extract_drawings(pdf_path, page_number=0):
    doc = fitz.open(pdf_path)
    page = doc.load_page(page_number)
    drawings = page.get_drawings()
    return drawings
```

### 2. **Identify Shapes Based on Geometric Properties**

Once we have the drawings, we can identify shapes based on their geometric properties:

* **Circles and Ovals**: These are typically represented by Bézier curves. We can approximate a circle or oval by checking if the shape has four Bézier curves and if the bounding box is approximately square (for circles) or rectangular (for ovals).

* **Rectangles**: These are usually represented by four straight lines forming a closed path. We can identify rectangles by checking if the shape has four line segments and if the angles between consecutive lines are approximately 90 degrees.

```python
def identify_shapes(drawings):
    shapes = []
    for drawing in drawings:
        if drawing["type"] == "curve" and len(drawing["items"]) == 4:
            # Likely a circle or oval
            bbox = drawing["rect"]
            aspect_ratio = bbox.width / bbox.height
            if 0.9 <= aspect_ratio <= 1.1:
                shapes.append(("circle", bbox))
            else:
                shapes.append(("oval", bbox))
        elif drawing["type"] == "line" and len(drawing["items"]) == 4:
            # Likely a rectangle
            shapes.append(("rectangle", drawing["rect"]))
    return shapes
```

### 3. **Extract Text Inside Identified Shapes**

After identifying the shapes, we can extract text that lies within their bounding boxes. PyMuPDF allows us to retrieve text blocks and check if their bounding boxes intersect with the identified shapes.

```python
def extract_text_inside_shapes(pdf_path, shapes, page_number=0):
    doc = fitz.open(pdf_path)
    page = doc.load_page(page_number)
    text_blocks = page.get_text("dict")["blocks"]
    extracted_text = []

    for shape_type, shape_bbox in shapes:
        for block in text_blocks:
            if block["type"] == 0:  # Text block
                for line in block["lines"]:
                    for span in line["spans"]:
                        span_bbox = fitz.Rect(span["bbox"])
                        if span_bbox.intersects(shape_bbox):
                            extracted_text.append(span["text"])

    return extracted_text
```

### 4. **Handle Rotated Shapes**

If shapes are rotated (e.g., by 90 degrees), their bounding boxes will still be axis-aligned, but the content inside might be rotated. To handle this, we can rotate the bounding boxes and the text blocks accordingly before checking for intersections.

```python
import math

def rotate_bbox(bbox, angle, center):
    angle_rad = math.radians(angle)
    cos_angle = math.cos(angle_rad)
    sin_angle = math.sin(angle_rad)

    x0, y0 = bbox.tl
    x1, y1 = bbox.br

    x0_rot = cos_angle * (x0 - center.x) - sin_angle * (y0 - center.y) + center.x
    y0_rot = sin_angle * (x0 - center.x) + cos_angle * (y0 - center.y) + center.y
    x1_rot = cos_angle * (x1 - center.x) - sin_angle * (y1 - center.y) + center.x
    y1_rot = sin_angle * (x1 - center.x) + cos_angle * (y1 - center.y) + center.y

    return fitz.Rect(x0_rot, y0_rot, x1_rot, y1_rot)

def extract_text_inside_rotated_shapes(pdf_path, shapes, page_number=0):
    doc = fitz.open(pdf_path)
    page = doc.load_page(page_number)
    text_blocks = page.get_text("dict")["blocks"]
    extracted_text = []

    for shape_type, shape_bbox in shapes:
        # Assuming the rotation angle is known or can be determined
        rotation_angle = 90  # Example: 90 degrees
        center = shape_bbox.center
        rotated_bbox = rotate_bbox(shape_bbox, rotation_angle, center)

        for block in text_blocks:
            if block["type"] == 0:  # Text block
                for line in block["lines"]:
                    for span in line["spans"]:
                        span_bbox = fitz.Rect(span["bbox"])
                        if span_bbox.intersects(rotated_bbox):
                            extracted_text.append(span["text"])

    return extracted_text
```

---

## 📊 Extracting Table Content

Extracting tables from PDFs can be challenging due to the lack of explicit table structures. However, we can attempt to extract tabular data by analyzing the positions and alignments of text blocks.

```python
def extract_table_content(pdf_path, page_number=0):
    doc = fitz.open(pdf_path)
    page = doc.load_page(page_number)
    text_blocks = page.get_text("dict")["blocks"]
    table_data = []

    for block in text_blocks:
        if block["type"] == 0:  # Text block
            row = []
            for line in block["lines"]:
                for span in line["spans"]:
                    row.append(span["text"])
            table_data.append(row)

    return table_data
```

---

## 🧪 Putting It All Together

```python
pdf_path = "your_file.pdf"
page_number = 0

# Step 1: Extract drawings
drawings = extract_drawings(pdf_path, page_number)

# Step 2: Identify shapes
shapes = identify_shapes(drawings)

# Step 3: Extract text inside shapes
extracted_text = extract_text_inside_shapes(pdf_path, shapes, page_number)

# Step 4: Extract table content
table_data = extract_table_content(pdf_path, page_number)

# Output results
print("Extracted Text Inside Shapes:")
for text in extracted_text:
    print(text)

print("\nExtracted Table Data:")
for row in table_data:
    print(row)
```

---

## ✅ Key Considerations

* **Tilted Shapes**: The provided code includes a method to handle rotated shapes by adjusting the bounding boxes accordingly.
* **Accuracy**: The accuracy of text extraction depends on the precision of shape detection and the alignment of text within those shapes.
* **Complex Tables**: For more complex tables, additional logic may
