---
.
.
.
.
.
.
.
.
.
.
.
.
.
.
.
.
.
.
.
.
.
.
---




Perfect! Let’s extend the solution in **two directions**:

---

## ✅ Extension 1: **Process All Pages of the PDF**

We’ll loop through every page, render it, detect shapes, and extract the text.

---

### 🧰 Updated Full Workflow – Multi-page PDF with Shape Detection

```python
import fitz
import cv2
import numpy as np
import os

def render_pdf_page_to_image(pdf_path, page_number=0, zoom=2):
    doc = fitz.open(pdf_path)
    page = doc.load_page(page_number)
    mat = fitz.Matrix(zoom, zoom)
    pix = page.get_pixmap(matrix=mat)
    image_path = f"page_{page_number + 1}.png"
    pix.save(image_path)
    return image_path, zoom

def detect_shapes(image_path):
    img = cv2.imread(image_path)
    gray = cv2.cvtColor(img, cv2.COLOR_BGR2GRAY)
    blurred = cv2.medianBlur(gray, 5)
    edges = cv2.Canny(blurred, 50, 150)

    contours, _ = cv2.findContours(edges, cv2.RETR_EXTERNAL, cv2.CHAIN_APPROX_SIMPLE)
    bounding_boxes = []

    for cnt in contours:
        approx = cv2.approxPolyDP(cnt, 0.02 * cv2.arcLength(cnt, True), True)
        x, y, w, h = cv2.boundingRect(approx)
        if 50 < w < 500 and 50 < h < 500:
            bounding_boxes.append((x, y, x + w, y + h))
    return bounding_boxes, img, contours

def scale_boxes_to_pdf(bounding_boxes, zoom):
    return [(x0 / zoom, y0 / zoom, x1 / zoom, y1 / zoom) for (x0, y0, x1, y1) in bounding_boxes]

def extract_text_in_rects(doc, rect_bboxes, page_number):
    page = doc.load_page(page_number)
    text_blocks = page.get_text("dict")["blocks"]
    results = []

    for rect in rect_bboxes:
        collected_text = ""
        for block in text_blocks:
            if block["type"] == 0:
                for line in block["lines"]:
                    for span in line["spans"]:
                        mid_x = (span["bbox"][0] + span["bbox"][2]) / 2
                        mid_y = (span["bbox"][1] + span["bbox"][3]) / 2
                        x0, y0, x1, y1 = rect
                        if x0 <= mid_x <= x1 and y0 <= mid_y <= y1:
                            collected_text += span["text"] + " "
        if collected_text:
            results.append(collected_text.strip())
    return results

def process_pdf_with_shapes(pdf_path):
    doc = fitz.open(pdf_path)
    all_results = {}

    for page_number in range(len(doc)):
        print(f"\nProcessing Page {page_number + 1}...")
        image_path, zoom = render_pdf_page_to_image(pdf_path, page_number)
        bounding_boxes, img, contours = detect_shapes(image_path)
        pdf_boxes = scale_boxes_to_pdf(bounding_boxes, zoom)
        texts = extract_text_in_rects(doc, pdf_boxes, page_number)

        all_results[page_number + 1] = texts

        # Draw bounding boxes and save image for verification
        for (x, y, x1, y1) in bounding_boxes:
            cv2.rectangle(img, (x, y), (x1, y1), (0, 255, 0), 2)
        output_img_path = f"annotated_page_{page_number + 1}.png"
        cv2.imwrite(output_img_path, img)
        print(f"Saved shape-annotated image: {output_img_path}")

        # Clean up rendered page image
        os.remove(image_path)

    return all_results
```

---

### 🧪 Use It:

```python
pdf_path = "your_autocad_file.pdf"
results = process_pdf_with_shapes(pdf_path)

for page_num, texts in results.items():
    print(f"\n📄 Page {page_num}:")
    for i, t in enumerate(texts, 1):
        print(f" [{i}] {t}")
```

---

## 🖼️ Output:

* The script generates `annotated_page_1.png`, `annotated_page_2.png`, etc.
* Each image will highlight detected shapes (rounded rectangles, circles).
* You also get extracted text per shape and page.

---

## 🚀 What Next?

Would you like to:

1. **Save the extracted data to a CSV or JSON?**
2. **Use OCR (e.g., Tesseract) in case some symbols are not recognized as text?**
3. **Improve shape detection for specific cases like half-circles or odd shapes?**

Let me know which path you'd like to explore further!

==================================================================
==================================================================
==================================================================

input query: 

```text
let us say in my pdf, I have some ovals in my pdf, inside that ovals, some content is there. 
also some circles are existed, inside circles some content, 
some times the content is plain text and some times it can be even symbols like half circle or half square or some kind of symbols. 
might be a different font not sure about pdf. so I need to extract content inside that circles and ovals. is it possible.
the circles are not placed in fixed location of page. but circles are of almost same size mostly. 
```

Thanks for the detailed explanation—that helps a lot. Based on what you've described, **yes, it's possible** to extract content from **inside ovals or circles** in a PDF using **PyMuPDF**, but there are some nuances and steps involved:

---

### ✅ **What You Can Do**

You can use **PyMuPDF** to:

1. **Extract all text blocks and their positions** (bounding boxes).
2. **Identify which blocks fall inside a circular or oval region**.
3. **Optionally detect vector shapes** like circles/ovals if they’re stored as drawing instructions (lines, curves).
4. **Handle special symbols**—though you might need to identify fonts or use unicode handling for non-text glyphs.

---

### 🔍 **How It Works Conceptually**

1. **Parse all shapes and text spans** on a page.
2. **Identify the bounding boxes of circles/ovals** (if extractable as shapes).
3. **For each shape**, check which text elements' centers fall **inside the ellipse/circle boundary**.
4. **Group and return the text** found within each shape.

---

### 🧪 **Step-by-Step Sample Code**

This code assumes you can visually or heuristically identify circle/oval **positions** (e.g., fixed radius or bounding boxes), and then match text inside:

```python
import fitz  # PyMuPDF
import math

def is_point_inside_ellipse(x, y, bbox):
    """Check if a point (x, y) is inside an elliptical bounding box."""
    x0, y0, x1, y1 = bbox
    cx, cy = (x0 + x1) / 2, (y0 + y1) / 2
    rx, ry = (x1 - x0) / 2, (y1 - y0) / 2
    return ((x - cx) ** 2) / (rx ** 2) + ((y - cy) ** 2) / (ry ** 2) <= 1

def extract_text_in_shapes(pdf_path, shape_bboxes):
    doc = fitz.open(pdf_path)
    results = []

    for page in doc:
        page_results = []
        text_blocks = page.get_text("dict")["blocks"]
        
        for bbox in shape_bboxes:
            collected_text = ""
            for block in text_blocks:
                if block["type"] == 0:  # Text block
                    for line in block["lines"]:
                        for span in line["spans"]:
                            x, y = span["bbox"][0], span["bbox"][1]  # top-left
                            mid_x = (span["bbox"][0] + span["bbox"][2]) / 2
                            mid_y = (span["bbox"][1] + span["bbox"][3]) / 2
                            if is_point_inside_ellipse(mid_x, mid_y, bbox):
                                collected_text += span["text"] + " "
            if collected_text:
                page_results.append(collected_text.strip())
        results.append(page_results)
    
    return results

# Sample usage with hypothetical circular/oval bounding boxes
pdf_file = "your_file.pdf"
# Example bounding boxes for oval regions (x0, y0, x1, y1)
shape_bounding_boxes = [
    (100, 150, 200, 250),  # Oval 1
    (300, 400, 400, 500),  # Oval 2
    # Add more as needed
]

extracted_text = extract_text_in_shapes(pdf_file, shape_bounding_boxes)
for i, page_texts in enumerate(extracted_text):
    print(f"Page {i + 1}:")
    for content in page_texts:
        print(f" - {content}")
```

---

### 🤖 If You Don't Know the Circle Positions

If the **ovals/circles are vector shapes**, PyMuPDF *might* expose them through `page.get_drawings()`:

```python
shapes = page.get_drawings()
for shape in shapes:
    if shape["type"] == "curve":  # Often ovals are curves
        print(shape["rect"], shape["items"])
```

But this is limited—you’d need to interpret the shapes heuristically (e.g., circular curves with similar width/height).

For **automated detection** of circles, you may need to:

* Render the PDF to an image (e.g., with PyMuPDF or pdf2image).
* Use **OpenCV** to detect circular shapes (`cv2.HoughCircles()`).
* Map the image coordinates back to PDF coordinates.

---

### 🧠 Special Symbols & Fonts

Some symbols (like half-circles, boxes, etc.) may:

* Appear as special Unicode characters.
* Be drawn as vector paths, not text.
* Use custom fonts or glyphs.

In such cases:

* Inspect span `["font"]` and `["text"]` in the output.
* Consider exporting spans with `get_text("dict")` and checking for non-ASCII or symbolic glyphs.
* Use OCR (e.g., Tesseract) on rendered images as fallback.

---

### 🔚 Summary

* ✅ Yes, extracting text inside circular/oval shapes is feasible.
* 🛠 You'll need to:

  * Detect those shapes (either manually, from drawings, or with computer vision).
  * Check which text blocks fall inside them.
* ⚠️ Special symbols may require extra font decoding or OCR.

Would you like help with a full version that includes OpenCV detection for auto-detecting the circle positions from the rendered PDF page?

Yes, absolutely — it's **okay** if the shapes are **rounded rectangles** instead of perfect ovals or circles. The good news is:

### ✅ Rounded Rectangles = Just Rectangles for Extraction Purposes

From PyMuPDF's perspective (and for most practical use cases), **a rounded rectangle behaves like a regular rectangle** when it comes to content positioning and bounding boxes.

---

## 🔄 What Changes?

* **Ellipse check (`is_point_inside_ellipse`)** can be **replaced with a simple rectangle containment check**.
* You just need to check whether the center of each text span is **within a rectangular bounding box**.

---

## 🔁 Updated Code for Rectangular or Rounded Rectangular Regions

Here’s an updated version of the earlier code tailored to **rectangles (including rounded ones):**

```python
import fitz  # PyMuPDF

def is_point_inside_rect(x, y, rect):
    """Check if a point is inside a rectangular area."""
    x0, y0, x1, y1 = rect
    return x0 <= x <= x1 and y0 <= y <= y1

def extract_text_in_rects(pdf_path, rect_bboxes):
    doc = fitz.open(pdf_path)
    results = []

    for page in doc:
        page_results = []
        text_blocks = page.get_text("dict")["blocks"]
        
        for rect in rect_bboxes:
            collected_text = ""
            for block in text_blocks:
                if block["type"] == 0:  # Text block
                    for line in block["lines"]:
                        for span in line["spans"]:
                            # Midpoint of text span box
                            mid_x = (span["bbox"][0] + span["bbox"][2]) / 2
                            mid_y = (span["bbox"][1] + span["bbox"][3]) / 2
                            if is_point_inside_rect(mid_x, mid_y, rect):
                                collected_text += span["text"] + " "
            if collected_text:
                page_results.append(collected_text.strip())
        results.append(page_results)
    
    return results

# Example usage
pdf_file = "your_file.pdf"
# Define some example rounded/regular rectangle bounding boxes
rounded_rects = [
    (100, 150, 200, 250),
    (300, 400, 400, 500),
]

text_inside_rects = extract_text_in_rects(pdf_file, rounded_rects)

for i, texts in enumerate(text_inside_rects):
    print(f"Page {i + 1}:")
    for content in texts:
        print(f" - {content}")
```

---

## 🧠 Pro Tip:

If you can't predefine the coordinates of those rounded rectangles:

* Use **`page.get_drawings()`** to try to identify them.
* Or render the page to an image and use **OpenCV's contour or rectangle detection**.

---

So yes—**rounded rectangles are totally fine** and possibly even easier to handle than ellipses.

Would you like to explore automatic shape detection with OpenCV from a rendered PDF page image?


==================================================================
==================================================================
==================================================================
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


==================================================================
==================================================================
==================================================================
Yes, absolutely — even **without converting the PDF to an image and using OpenCV**, you can still extract a **significant amount of content directly** using **PyMuPDF** alone.

Here’s a breakdown of what **you can still do directly** with PyMuPDF:

---

## ✅ What *Can* Be Extracted Directly

### 1. **Text Content**

* PyMuPDF (`page.get_text("dict")`) gives:

  * Words, lines, blocks
  * Bounding boxes for each span of text
* You can check **if text falls inside known regions** (like where you expect circles or rectangles).

### 2. **Text Formatting Info**

* Fonts, size, color, encoding, and even some special Unicode glyphs (like symbols).
* Helpful to distinguish between regular text and symbolic fonts.

### 3. **Annotations (if any)**

* If the document has annotations (highlighted or popup-style), those can be extracted with `.annots()`.

### 4. **Drawings / Shapes**

* You can access vector drawing instructions (e.g., lines, curves, circles) using `page.get_drawings()`.

---

## ❌ What You *Can’t Easily* Detect Without OpenCV

### 1. **Implicit Shapes**

* If the circles/ovals are **pure vector graphics without metadata**, PyMuPDF may return them as raw paths — not labeled as “this is a circle.”
* You'll see something like:

  ```python
  [{'type': 'curve', 'items': [...], 'rect': (x0, y0, x1, y1)}]
  ```

  but you’ll have to **manually analyze path items** to know if it’s a circle or oval.

### 2. **Symbols as Vector Paths**

* If symbols (like half-squares) are **not real text**, but vector paths or embedded fonts, PyMuPDF won't extract them as readable characters unless the font is properly embedded and readable.

---

## ✅ Recommended Direct Approach (Without OpenCV)

Here’s what you can still do effectively:

1. **Extract all text blocks**
2. **Use heuristics** (e.g. “if text block is small and positioned similarly to where I expect a circle, capture it”)
3. **Optionally filter by font or symbols**

### ✅ Example (no OpenCV):

```python
import fitz

def extract_candidate_text_blocks(pdf_path):
    doc = fitz.open(pdf_path)
    all_blocks = []

    for page_num in range(len(doc)):
        page = doc.load_page(page_num)
        text_dict = page.get_text("dict")

        for block in text_dict["blocks"]:
            if block["type"] == 0:
                for line in block["lines"]:
                    for span in line["spans"]:
                        text = span["text"]
                        bbox = span["bbox"]
                        font = span["font"]
                        size = span["size"]

                        # Filter: small text blocks could be inside circles
                        if len(text.strip()) <= 5 and 6 < size < 15:
                            all_blocks.append({
                                "page": page_num + 1,
                                "text": text,
                                "bbox": bbox,
                                "font": font,
                                "size": size,
                            })
    return all_blocks

results = extract_candidate_text_blocks("your_file.pdf")
for item in results:
    print(f"[Page {item['page']}] {item['text']} at {item['bbox']} (font: {item['font']})")
```

---

### 🔍 Tip:

You can add logic like:

* **Check if the `bbox` is roughly square** (might be a symbol inside a circle).
* **Check font name** — sometimes symbolic fonts have custom names like `Wingdings`, `Symbol`, or `CustomFont123`.

---

### 🔚 Summary

✔️ You *can still extract content* directly without OpenCV.
✔️ Text (and sometimes even symbolic glyphs) can be pulled using `get_text("dict")`.
🚫 You won’t get shape detection (like detecting circles) unless:

* You define shape positions yourself
* Or use shape metadata (if available)

Would you like help writing logic to **auto-detect square-ish text boxes** as a proxy for symbols inside circles?


==================================================================
==================================================================
==================================================================
Let's delve deeper into extracting layered content from your AutoCAD-generated PDFs using PyMuPDF. AutoCAD PDFs often utilize Optional Content Groups (OCGs), which allow for the inclusion of multiple layers within a single PDF. These layers can represent different sets of information, such as dimensions, annotations, or different language versions. PyMuPDF provides tools to interact with these layers effectively.([GitHub][1])

---

## 🔍 Understanding Optional Content Groups (OCGs)

OCGs, also known as layers, are a feature in PDFs that enable content to be grouped and optionally displayed or hidden. Each OCG has a visibility state and can be toggled on or off. In AutoCAD PDFs, these layers might correspond to different types of content, like annotations, dimensions, or background graphics.([Artifex][2])

---

## 🧩 Step-by-Step Guide to Extract Layered Content

### 1. **Check for Available Layers**

First, determine if your PDF contains any OCGs:

```python
import fitz  # PyMuPDF

# Open the PDF
doc = fitz.open("your_autocad_file.pdf")

# Retrieve the layer UI configurations
layer_configs = doc.layer_ui_configs()

# Check if there are any layers
if not layer_configs:
    print("No layers found in this PDF.")
else:
    print(f"Found {len(layer_configs)} layers:")
    for layer in layer_configs:
        print(f"Layer: {layer['text']}, Visible: {layer['on']}")
```



This script will list all available layers and their current visibility states.([Artifex][2])

### 2. **Toggle Layer Visibility**

To extract content from a specific layer, you can toggle its visibility:([Artifex][2])

```python
# Turn off the 'english' layer (replace with your layer's name)
doc.set_layer_ui_config(1, action=2)  # action=2 sets the layer to OFF

# Extract text from the page
page = doc.load_page(0)  # Load the first page
text = page.get_text("text")
print(text)
```



This will extract text from the first page with the 'english' layer turned off. Adjust the layer number as needed.([Artifex][2])

### 3. **Extract Annotations by Layer**

Annotations in PDFs can also be associated with specific layers. To extract annotations by their associated layer:([Stack Overflow][3])

```python
from collections import defaultdict

# Initialize a dictionary to store annotations by layer
annotations_by_layer = defaultdict(list)

# Iterate through each page
for page_num in range(len(doc)):
    page = doc.load_page(page_num)
    for annot in page.annots():
        # Get the layer (OCG) associated with the annotation
        ocg_xref = annot.get_oc()
        if ocg_xref > 0:
            ocg_dict = doc.get_ocgs()[ocg_xref]
            layer_name = ocg_dict["name"]
            annotations_by_layer[layer_name].append(annot.rect)

# Print annotations grouped by layer
for layer, annotations in annotations_by_layer.items():
    print(f"Layer: {layer}, Annotations: {annotations}")
```



This script groups annotations by their associated layer and prints their bounding boxes.

---

## 🧠 Tips for Working with Layers in AutoCAD PDFs

* **Layer Visibility**: Use `doc.layer_ui_configs()` to inspect and manage layer visibility.
* **Layer-Specific Extraction**: Toggle layers on or off using `doc.set_layer_ui_config()` to extract content specific to a layer.
* **Annotations**: Annotations can be associated with specific layers. Use `annot.get_oc()` to retrieve the OCG (Optional Content Group) associated with an annotation.
* **Performance**: Be mindful of performance when dealing with large PDFs or numerous layers. Consider processing pages in batches or using asynchronous methods if available.([Artifex][2], [Stack Overflow][3])

---

## 📚 Further Reading

For more detailed information and advanced techniques, refer to the following resources:

* **PyMuPDF Documentation**: Comprehensive guide to PyMuPDF's features and capabilities.
* **Artifex Blog on PDF Layers**: In-depth discussion on accessing and manipulating PDF layers using PyMuPDF.
* **Stack Overflow Discussion**: Community-driven solutions and examples for extracting annotations by layer.([Stack Overflow][3])

If you have specific requirements or encounter any issues while working with your AutoCAD PDFs, feel free to provide more details, and I'll be glad to assist further.

[1]: https://github.com/pymupdf/PyMuPDF/discussions/4091?utm_source=chatgpt.com "Challenges with Extracting and Reapplying Specific Layers from Multi-layer PDFs using PyMuPDF · pymupdf PyMuPDF · Discussion #4091 · GitHub"
[2]: https://artifex.com/blog/optional-content-discovering-the-pdf-layers-pymupdf-python?utm_source=chatgpt.com "Blog - Artifex"
[3]: https://stackoverflow.com/questions/77066094/extract-annotations-by-layer-from-a-pdf-in-python?utm_source=chatgpt.com "Extract annotations by layer from a PDF in Python - Stack Overflow"


==================================================================
==================================================================
==================================================================
Great! Let's walk through a full solution where we:

1. **Render the PDF page as an image** using PyMuPDF.
2. **Use OpenCV to detect rounded rectangles, circles, or ovals**.
3. **Convert the detected image coordinates back to PDF coordinates**.
4. **Extract text from those detected regions** using PyMuPDF.

---

## 🧰 Libraries Required

```bash
pip install pymupdf opencv-python
```

---

## 📷 Step 1: Render PDF Page to Image

```python
import fitz  # PyMuPDF

def render_pdf_page_to_image(pdf_path, page_number=0, zoom=2):
    doc = fitz.open(pdf_path)
    page = doc.load_page(page_number)
    mat = fitz.Matrix(zoom, zoom)  # zoom > 1 = higher resolution
    pix = page.get_pixmap(matrix=mat)
    image_path = f"page_{page_number + 1}.png"
    pix.save(image_path)
    return image_path, zoom
```

---

## 🔍 Step 2: Detect Rounded Rectangles or Circles Using OpenCV

```python
import cv2
import numpy as np

def detect_shapes(image_path):
    img = cv2.imread(image_path)
    gray = cv2.cvtColor(img, cv2.COLOR_BGR2GRAY)
    blurred = cv2.medianBlur(gray, 5)
    edges = cv2.Canny(blurred, 50, 150)

    contours, _ = cv2.findContours(edges, cv2.RETR_EXTERNAL, cv2.CHAIN_APPROX_SIMPLE)

    bounding_boxes = []

    for cnt in contours:
        approx = cv2.approxPolyDP(cnt, 0.02 * cv2.arcLength(cnt, True), True)
        x, y, w, h = cv2.boundingRect(approx)

        # You can add filtering logic here (e.g. min size, aspect ratio)
        if 50 < w < 500 and 50 < h < 500:  # basic filter
            bounding_boxes.append((x, y, x + w, y + h))

    return bounding_boxes
```

---

## 🔁 Step 3: Convert Image Coords Back to PDF Space

PyMuPDF zooms the image, so to get back to **PDF coordinate space**:

```python
def scale_boxes_to_pdf(bounding_boxes, zoom):
    pdf_boxes = []
    for x0, y0, x1, y1 in bounding_boxes:
        pdf_box = (x0 / zoom, y0 / zoom, x1 / zoom, y1 / zoom)
        pdf_boxes.append(pdf_box)
    return pdf_boxes
```

---

## 📄 Step 4: Extract Text Inside Detected Shapes

We use the same rectangular text extraction function from earlier:

```python
def extract_text_in_rects(pdf_path, rect_bboxes, page_number=0):
    doc = fitz.open(pdf_path)
    page = doc.load_page(page_number)
    text_blocks = page.get_text("dict")["blocks"]
    results = []

    for rect in rect_bboxes:
        collected_text = ""
        for block in text_blocks:
            if block["type"] == 0:
                for line in block["lines"]:
                    for span in line["spans"]:
                        mid_x = (span["bbox"][0] + span["bbox"][2]) / 2
                        mid_y = (span["bbox"][1] + span["bbox"][3]) / 2
                        x0, y0, x1, y1 = rect
                        if x0 <= mid_x <= x1 and y0 <= mid_y <= y1:
                            collected_text += span["text"] + " "
        if collected_text:
            results.append(collected_text.strip())
    return results
```

---

## 🧪 Complete Workflow Example

```python
pdf_file = "your_autocad_file.pdf"
page_number = 0

# Step 1: Render page as image
image_path, zoom = render_pdf_page_to_image(pdf_file, page_number)

# Step 2: Detect rounded rectangles or circles
detected_shapes = detect_shapes(image_path)

# Step 3: Convert image boxes to PDF coordinate space
pdf_boxes = scale_boxes_to_pdf(detected_shapes, zoom)

# Step 4: Extract text inside those regions
texts = extract_text_in_rects(pdf_file, pdf_boxes, page_number)

# Output
print("Detected text inside rounded shapes:")
for idx, t in enumerate(texts):
    print(f"[{idx + 1}] {t}")
```

---

## 📌 Notes

* You can tweak the **`detect_shapes`** function to be more aggressive or precise by using techniques like:

  * `cv2.HoughCircles()` for circular shape detection.
  * Checking contour aspect ratio and extent for rounded rectangles.
* For more complex glyphs (e.g. symbols), you can detect shape **and then OCR the region** if needed.

---


==================================================================
==================================================================
==================================================================
PyMuPDF (also known as `fitz`) is a powerful Python library for working with PDF files. It allows you to extract text, images, and metadata, as well as manipulate and analyze the content of PDFs. This is particularly useful when dealing with complex documents like AutoCAD-generated PDFs, which may include vector graphics, annotations, and layered content.([devgem.io][1])

---

## 🔧 What You Can Do with PyMuPDF

### 1. **Extract Text**

You can extract text from PDF pages in various formats:([Artifex][2])

* **Plain Text**: Extracts raw text without any formatting.
* **Blocks**: Returns text as blocks, useful for structured documents.
* **HTML**: Extracts text with basic HTML formatting.
* **JSON**: Provides detailed information about the text layout.([CodeEase][3])

For example, to extract plain text:

```python
import fitz  # PyMuPDF

doc = fitz.open("your_file.pdf")
for page in doc:
    text = page.get_text("text")
    print(text)
```



This will print the extracted text from each page.

### 2. **Extract Images**

You can extract images embedded in the PDF:

```python
import fitz

doc = fitz.open("your_file.pdf")
for page_num in range(len(doc)):
    page = doc.load_page(page_num)
    image_list = page.get_images(full=True)
    for img_index, img in enumerate(image_list):
        xref = img[0]
        base_image = doc.extract_image(xref)
        image_bytes = base_image["image"]
        with open(f"image{page_num + 1}_{img_index + 1}.png", "wb") as img_file:
            img_file.write(image_bytes)
```



This script saves each image as a PNG file.

### 3. **Extract Metadata**

Retrieve metadata such as author, title, and creation date:([etutorialspoint.com][4])

```python
import fitz

doc = fitz.open("your_file.pdf")
metadata = doc.metadata
print(metadata)
```



This will print the metadata of the PDF.

### 4. **Extract Tables**

PyMuPDF can identify and extract tables from PDFs:([Artifex][2])

```python
import fitz

doc = fitz.open("your_file.pdf")
for page in doc:
    blocks = page.get_text("dict")["blocks"]
    for b in blocks:
        if b["type"] == 0:  # Text block
            print(b["text"])
```



This example prints text blocks, which can be further processed to identify tables.

### 5. **Extract Layout Information**

For more detailed analysis, you can extract layout information:

```python
import fitz

doc = fitz.open("your_file.pdf")
for page in doc:
    blocks = page.get_text("dict")["blocks"]
    for b in blocks:
        if b["type"] == 0:  # Text block
            for line in b["lines"]:
                for span in line["spans"]:
                    print(f"Text: {span['text']}, Font: {span['font']}, Size: {span['size']}")
```



This provides detailed information about each text span, including font and size.([Medium][5])

---

## 🧭 PyMuPDF Documentation Highlights

The PyMuPDF tutorial provides comprehensive guidance on:

* **Opening and Navigating PDFs**: Learn how to open PDF files and navigate through pages.
* **Extracting Text and Images**: Detailed methods for extracting text and images from PDFs.
* **Working with Metadata**: Accessing and manipulating PDF metadata.
* **Modifying PDFs**: Techniques for editing and creating PDFs.
* **Advanced Features**: Utilizing PyMuPDF's advanced features for complex PDF manipulation.

For AutoCAD PDFs, which may contain vector graphics and annotations, PyMuPDF's ability to handle complex layouts and extract detailed information makes it a suitable choice. You can use the `get_text("dict")` method to analyze the structure of the document and extract relevant content.([Welcome to python-forum.io][6])

---

## ✅ Sample Workflow for AutoCAD PDFs

Here's a sample workflow to extract and process content from an AutoCAD-generated PDF:

```python
import fitz

doc = fitz.open("your_autocad_file.pdf")
for page_num in range(len(doc)):
    page = doc.load_page(page_num)
    blocks = page.get_text("dict")["blocks"]
    for b in blocks:
        if b["type"] == 0:  # Text block
            print(f"Page {page_num + 1}: {b['text']}")
        elif b["type"] == 1:  # Image block
            xref = b["image"]
            base_image = doc.extract_image(xref)
            image_bytes = base_image["image"]
            with open(f"image{page_num + 1}.png", "wb") as img_file:
                img_file.write(image_bytes)
```



This script processes each page, extracting text and images, and saves the images as PNG files.

---

## 🧪 Tips for Handling AutoCAD PDFs

* **Layered Content**: AutoCAD PDFs may have layered content. Use PyMuPDF's `get_text("dict")` method to analyze and extract specific layers.
* **Vector Graphics**: For vector graphics, PyMuPDF's image extraction capabilities can be utilized, though handling complex vector data may require additional processing.
* **Annotations**: Annotations such as dimensions and notes can be extracted using the `get_text("dict")` method, which provides detailed information about text spans.([Welcome to python-forum.io][6])

---

## 📚 Learn More

For a deeper understanding and more advanced techniques, refer to the [PyMuPDF Tutorial](https://pymupdf.readthedocs.io/en/latest/tutorial.html). It offers detailed explanations and examples to help you effectively work with PDF documents.

If you need assistance

[1]: https://www.devgem.io/posts/extracting-clean-text-from-pdf-using-pymupdf-a-comprehensive-guide?utm_source=chatgpt.com "Extracting Clean Text from PDF using PyMuPDF: A Comprehensive Guide – devgem.io - devgem.io"
[2]: https://artifex.com/blog/table-recognition-extraction-from-pdfs-pymupdf-python?utm_source=chatgpt.com "Table Recognition and Extraction With ..."
[3]: https://codeease.net/programming/python/pymupdf-extract-allPerfect! Let’s extend the solution in **two directions**:

---

## ✅ Extension 1: **Process All Pages of the PDF**

We’ll loop through every page, render it, detect shapes, and extract the text.

---

### 🧰 Updated Full Workflow – Multi-page PDF with Shape Detection

```python
import fitz
import cv2
import numpy as np
import os

def render_pdf_page_to_image(pdf_path, page_number=0, zoom=2):
    doc = fitz.open(pdf_path)
    page = doc.load_page(page_number)
    mat = fitz.Matrix(zoom, zoom)
    pix = page.get_pixmap(matrix=mat)
    image_path = f"page_{page_number + 1}.png"
    pix.save(image_path)
    return image_path, zoom

def detect_shapes(image_path):
    img = cv2.imread(image_path)
    gray = cv2.cvtColor(img, cv2.COLOR_BGR2GRAY)
    blurred = cv2.medianBlur(gray, 5)
    edges = cv2.Canny(blurred, 50, 150)

    contours, _ = cv2.findContours(edges, cv2.RETR_EXTERNAL, cv2.CHAIN_APPROX_SIMPLE)
    bounding_boxes = []

    for cnt in contours:
        approx = cv2.approxPolyDP(cnt, 0.02 * cv2.arcLength(cnt, True), True)
        x, y, w, h = cv2.boundingRect(approx)
        if 50 < w < 500 and 50 < h < 500:
            bounding_boxes.append((x, y, x + w, y + h))
    return bounding_boxes, img, contours

def scale_boxes_to_pdf(bounding_boxes, zoom):
    return [(x0 / zoom, y0 / zoom, x1 / zoom, y1 / zoom) for (x0, y0, x1, y1) in bounding_boxes]

def extract_text_in_rects(doc, rect_bboxes, page_number):
    page = doc.load_page(page_number)
    text_blocks = page.get_text("dict")["blocks"]
    results = []

    for rect in rect_bboxes:
        collected_text = ""
        for block in text_blocks:
            if block["type"] == 0:
                for line in block["lines"]:
                    for span in line["spans"]:
                        mid_x = (span["bbox"][0] + span["bbox"][2]) / 2
                        mid_y = (span["bbox"][1] + span["bbox"][3]) / 2
                        x0, y0, x1, y1 = rect
                        if x0 <= mid_x <= x1 and y0 <= mid_y <= y1:
                            collected_text += span["text"] + " "
        if collected_text:
            results.append(collected_text.strip())
    return results

def process_pdf_with_shapes(pdf_path):
    doc = fitz.open(pdf_path)
    all_results = {}

    for page_number in range(len(doc)):
        print(f"\nProcessing Page {page_number + 1}...")
        image_path, zoom = render_pdf_page_to_image(pdf_path, page_number)
        bounding_boxes, img, contours = detect_shapes(image_path)
        pdf_boxes = scale_boxes_to_pdf(bounding_boxes, zoom)
        texts = extract_text_in_rects(doc, pdf_boxes, page_number)

        all_results[page_number + 1] = texts

        # Draw bounding boxes and save image for verification
        for (x, y, x1, y1) in bounding_boxes:
            cv2.rectangle(img, (x, y), (x1, y1), (0, 255, 0), 2)
        output_img_path = f"annotated_page_{page_number + 1}.png"
        cv2.imwrite(output_img_path, img)
        print(f"Saved shape-annotated image: {output_img_path}")

        # Clean up rendered page image
        os.remove(image_path)

    return all_results
```

---

### 🧪 Use It:

```python
pdf_path = "your_autocad_file.pdf"
results = process_pdf_with_shapes(pdf_path)

for page_num, texts in results.items():
    print(f"\n📄 Page {page_num}:")
    for i, t in enumerate(texts, 1):
        print(f" [{i}] {t}")
```

---

## 🖼️ Output:

* The script generates `annotated_page_1.png`, `annotated_page_2.png`, etc.
* Each image will highlight detected shapes (rounded rectangles, circles).
* You also get extracted text per shape and page.

---

## 🚀 What Next?

Would you like to:

1. **Save the extracted data to a CSV or JSON?**
2. **Use OCR (e.g., Tesseract) in case some symbols are not recognized as text?**
3. **Improve shape detection for specific cases like half-circles or odd shapes?**

Let me know which path you'd like to explore further!

==================================================================
==================================================================
==================================================================

input query: 

```text
let us say in my pdf, I have some ovals in my pdf, inside that ovals, some content is there. 
also some circles are existed, inside circles some content, 
some times the content is plain text and some times it can be even symbols like half circle or half square or some kind of symbols. 
might be a different font not sure about pdf. so I need to extract content inside that circles and ovals. is it possible.
the circles are not placed in fixed location of page. but circles are of almost same size mostly. 
```

Thanks for the detailed explanation—that helps a lot. Based on what you've described, **yes, it's possible** to extract content from **inside ovals or circles** in a PDF using **PyMuPDF**, but there are some nuances and steps involved:

---

### ✅ **What You Can Do**

You can use **PyMuPDF** to:

1. **Extract all text blocks and their positions** (bounding boxes).
2. **Identify which blocks fall inside a circular or oval region**.
3. **Optionally detect vector shapes** like circles/ovals if they’re stored as drawing instructions (lines, curves).
4. **Handle special symbols**—though you might need to identify fonts or use unicode handling for non-text glyphs.

---

### 🔍 **How It Works Conceptually**

1. **Parse all shapes and text spans** on a page.
2. **Identify the bounding boxes of circles/ovals** (if extractable as shapes).
3. **For each shape**, check which text elements' centers fall **inside the ellipse/circle boundary**.
4. **Group and return the text** found within each shape.

---

### 🧪 **Step-by-Step Sample Code**

This code assumes you can visually or heuristically identify circle/oval **positions** (e.g., fixed radius or bounding boxes), and then match text inside:

```python
import fitz  # PyMuPDF
import math

def is_point_inside_ellipse(x, y, bbox):
    """Check if a point (x, y) is inside an elliptical bounding box."""
    x0, y0, x1, y1 = bbox
    cx, cy = (x0 + x1) / 2, (y0 + y1) / 2
    rx, ry = (x1 - x0) / 2, (y1 - y0) / 2
    return ((x - cx) ** 2) / (rx ** 2) + ((y - cy) ** 2) / (ry ** 2) <= 1

def extract_text_in_shapes(pdf_path, shape_bboxes):
    doc = fitz.open(pdf_path)
    results = []

    for page in doc:
        page_results = []
        text_blocks = page.get_text("dict")["blocks"]
        
        for bbox in shape_bboxes:
            collected_text = ""
            for block in text_blocks:
                if block["type"] == 0:  # Text block
                    for line in block["lines"]:
                        for span in line["spans"]:
                            x, y = span["bbox"][0], span["bbox"][1]  # top-left
                            mid_x = (span["bbox"][0] + span["bbox"][2]) / 2
                            mid_y = (span["bbox"][1] + span["bbox"][3]) / 2
                            if is_point_inside_ellipse(mid_x, mid_y, bbox):
                                collected_text += span["text"] + " "
            if collected_text:
                page_results.append(collected_text.strip())
        results.append(page_results)
    
    return results

# Sample usage with hypothetical circular/oval bounding boxes
pdf_file = "your_file.pdf"
# Example bounding boxes for oval regions (x0, y0, x1, y1)
shape_bounding_boxes = [
    (100, 150, 200, 250),  # Oval 1
    (300, 400, 400, 500),  # Oval 2
    # Add more as needed
]

extracted_text = extract_text_in_shapes(pdf_file, shape_bounding_boxes)
for i, page_texts in enumerate(extracted_text):
    print(f"Page {i + 1}:")
    for content in page_texts:
        print(f" - {content}")
```

---

### 🤖 If You Don't Know the Circle Positions

If the **ovals/circles are vector shapes**, PyMuPDF *might* expose them through `page.get_drawings()`:

```python
shapes = page.get_drawings()
for shape in shapes:
    if shape["type"] == "curve":  # Often ovals are curves
        print(shape["rect"], shape["items"])
```

But this is limited—you’d need to interpret the shapes heuristically (e.g., circular curves with similar width/height).

For **automated detection** of circles, you may need to:

* Render the PDF to an image (e.g., with PyMuPDF or pdf2image).
* Use **OpenCV** to detect circular shapes (`cv2.HoughCircles()`).
* Map the image coordinates back to PDF coordinates.

---

### 🧠 Special Symbols & Fonts

Some symbols (like half-circles, boxes, etc.) may:

* Appear as special Unicode characters.
* Be drawn as vector paths, not text.
* Use custom fonts or glyphs.

In such cases:

* Inspect span `["font"]` and `["text"]` in the output.
* Consider exporting spans with `get_text("dict")` and checking for non-ASCII or symbolic glyphs.
* Use OCR (e.g., Tesseract) on rendered images as fallback.

---

### 🔚 Summary

* ✅ Yes, extracting text inside circular/oval shapes is feasible.
* 🛠 You'll need to:

  * Detect those shapes (either manually, from drawings, or with computer vision).
  * Check which text blocks fall inside them.
* ⚠️ Special symbols may require extra font decoding or OCR.

Would you like help with a full version that includes OpenCV detection for auto-detecting the circle positions from the rendered PDF page?

Yes, absolutely — it's **okay** if the shapes are **rounded rectangles** instead of perfect ovals or circles. The good news is:

### ✅ Rounded Rectangles = Just Rectangles for Extraction Purposes

From PyMuPDF's perspective (and for most practical use cases), **a rounded rectangle behaves like a regular rectangle** when it comes to content positioning and bounding boxes.

---

## 🔄 What Changes?

* **Ellipse check (`is_point_inside_ellipse`)** can be **replaced with a simple rectangle containment check**.
* You just need to check whether the center of each text span is **within a rectangular bounding box**.

---

## 🔁 Updated Code for Rectangular or Rounded Rectangular Regions

Here’s an updated version of the earlier code tailored to **rectangles (including rounded ones):**

```python
import fitz  # PyMuPDF

def is_point_inside_rect(x, y, rect):
    """Check if a point is inside a rectangular area."""
    x0, y0, x1, y1 = rect
    return x0 <= x <= x1 and y0 <= y <= y1

def extract_text_in_rects(pdf_path, rect_bboxes):
    doc = fitz.open(pdf_path)
    results = []

    for page in doc:
        page_results = []
        text_blocks = page.get_text("dict")["blocks"]
        
        for rect in rect_bboxes:
            collected_text = ""
            for block in text_blocks:
                if block["type"] == 0:  # Text block
                    for line in block["lines"]:
                        for span in line["spans"]:
                            # Midpoint of text span box
                            mid_x = (span["bbox"][0] + span["bbox"][2]) / 2
                            mid_y = (span["bbox"][1] + span["bbox"][3]) / 2
                            if is_point_inside_rect(mid_x, mid_y, rect):
                                collected_text += span["text"] + " "
            if collected_text:
                page_results.append(collected_text.strip())
        results.append(page_results)
    
    return results

# Example usage
pdf_file = "your_file.pdf"
# Define some example rounded/regular rectangle bounding boxes
rounded_rects = [
    (100, 150, 200, 250),
    (300, 400, 400, 500),
]

text_inside_rects = extract_text_in_rects(pdf_file, rounded_rects)

for i, texts in enumerate(text_inside_rects):
    print(f"Page {i + 1}:")
    for content in texts:
        print(f" - {content}")
```

---

## 🧠 Pro Tip:

If you can't predefine the coordinates of those rounded rectangles:

* Use **`page.get_drawings()`** to try to identify them.
* Or render the page to an image and use **OpenCV's contour or rectangle detection**.

---

So yes—**rounded rectangles are totally fine** and possibly even easier to handle than ellipses.

Would you like to explore automatic shape detection with OpenCV from a rendered PDF page image?


==================================================================
==================================================================
==================================================================
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


==================================================================
==================================================================
==================================================================
Yes, absolutely — even **without converting the PDF to an image and using OpenCV**, you can still extract a **significant amount of content directly** using **PyMuPDF** alone.

Here’s a breakdown of what **you can still do directly** with PyMuPDF:

---

## ✅ What *Can* Be Extracted Directly

### 1. **Text Content**

* PyMuPDF (`page.get_text("dict")`) gives:

  * Words, lines, blocks
  * Bounding boxes for each span of text
* You can check **if text falls inside known regions** (like where you expect circles or rectangles).

### 2. **Text Formatting Info**

* Fonts, size, color, encoding, and even some special Unicode glyphs (like symbols).
* Helpful to distinguish between regular text and symbolic fonts.

### 3. **Annotations (if any)**

* If the document has annotations (highlighted or popup-style), those can be extracted with `.annots()`.

### 4. **Drawings / Shapes**

* You can access vector drawing instructions (e.g., lines, curves, circles) using `page.get_drawings()`.

---

## ❌ What You *Can’t Easily* Detect Without OpenCV

### 1. **Implicit Shapes**

* If the circles/ovals are **pure vector graphics without metadata**, PyMuPDF may return them as raw paths — not labeled as “this is a circle.”
* You'll see something like:

  ```python
  [{'type': 'curve', 'items': [...], 'rect': (x0, y0, x1, y1)}]
  ```

  but you’ll have to **manually analyze path items** to know if it’s a circle or oval.

### 2. **Symbols as Vector Paths**

* If symbols (like half-squares) are **not real text**, but vector paths or embedded fonts, PyMuPDF won't extract them as readable characters unless the font is properly embedded and readable.

---

## ✅ Recommended Direct Approach (Without OpenCV)

Here’s what you can still do effectively:

1. **Extract all text blocks**
2. **Use heuristics** (e.g. “if text block is small and positioned similarly to where I expect a circle, capture it”)
3. **Optionally filter by font or symbols**

### ✅ Example (no OpenCV):

```python
import fitz

def extract_candidate_text_blocks(pdf_path):
    doc = fitz.open(pdf_path)
    all_blocks = []

    for page_num in range(len(doc)):
        page = doc.load_page(page_num)
        text_dict = page.get_text("dict")

        for block in text_dict["blocks"]:
            if block["type"] == 0:
                for line in block["lines"]:
                    for span in line["spans"]:
                        text = span["text"]
                        bbox = span["bbox"]
                        font = span["font"]
                        size = span["size"]

                        # Filter: small text blocks could be inside circles
                        if len(text.strip()) <= 5 and 6 < size < 15:
                            all_blocks.append({
                                "page": page_num + 1,
                                "text": text,
                                "bbox": bbox,
                                "font": font,
                                "size": size,
                            })
    return all_blocks

results = extract_candidate_text_blocks("your_file.pdf")
for item in results:
    print(f"[Page {item['page']}] {item['text']} at {item['bbox']} (font: {item['font']})")
```

---

### 🔍 Tip:

You can add logic like:

* **Check if the `bbox` is roughly square** (might be a symbol inside a circle).
* **Check font name** — sometimes symbolic fonts have custom names like `Wingdings`, `Symbol`, or `CustomFont123`.

---

### 🔚 Summary

✔️ You *can still extract content* directly without OpenCV.
✔️ Text (and sometimes even symbolic glyphs) can be pulled using `get_text("dict")`.
🚫 You won’t get shape detection (like detecting circles) unless:

* You define shape positions yourself
* Or use shape metadata (if available)

Would you like help writing logic to **auto-detect square-ish text boxes** as a proxy for symbols inside circles?


==================================================================
==================================================================
==================================================================
Let's delve deeper into extracting layered content from your AutoCAD-generated PDFs using PyMuPDF. AutoCAD PDFs often utilize Optional Content Groups (OCGs), which allow for the inclusion of multiple layers within a single PDF. These layers can represent different sets of information, such as dimensions, annotations, or different language versions. PyMuPDF provides tools to interact with these layers effectively.([GitHub][1])

---

## 🔍 Understanding Optional Content Groups (OCGs)

OCGs, also known as layers, are a feature in PDFs that enable content to be grouped and optionally displayed or hidden. Each OCG has a visibility state and can be toggled on or off. In AutoCAD PDFs, these layers might correspond to different types of content, like annotations, dimensions, or background graphics.([Artifex][2])

---

## 🧩 Step-by-Step Guide to Extract Layered Content

### 1. **Check for Available Layers**

First, determine if your PDF contains any OCGs:

```python
import fitz  # PyMuPDF

# Open the PDF
doc = fitz.open("your_autocad_file.pdf")

# Retrieve the layer UI configurations
layer_configs = doc.layer_ui_configs()

# Check if there are any layers
if not layer_configs:
    print("No layers found in this PDF.")
else:
    print(f"Found {len(layer_configs)} layers:")
    for layer in layer_configs:
        print(f"Layer: {layer['text']}, Visible: {layer['on']}")
```



This script will list all available layers and their current visibility states.([Artifex][2])

### 2. **Toggle Layer Visibility**

To extract content from a specific layer, you can toggle its visibility:([Artifex][2])

```python
# Turn off the 'english' layer (replace with your layer's name)
doc.set_layer_ui_config(1, action=2)  # action=2 sets the layer to OFF

# Extract text from the page
page = doc.load_page(0)  # Load the first page
text = page.get_text("text")
print(text)
```



This will extract text from the first page with the 'english' layer turned off. Adjust the layer number as needed.([Artifex][2])

### 3. **Extract Annotations by Layer**

Annotations in PDFs can also be associated with specific layers. To extract annotations by their associated layer:([Stack Overflow][3])

```python
from collections import defaultdict

# Initialize a dictionary to store annotations by layer
annotations_by_layer = defaultdict(list)

# Iterate through each page
for page_num in range(len(doc)):
    page = doc.load_page(page_num)
    for annot in page.annots():
        # Get the layer (OCG) associated with the annotation
        ocg_xref = annot.get_oc()
        if ocg_xref > 0:
            ocg_dict = doc.get_ocgs()[ocg_xref]
            layer_name = ocg_dict["name"]
            annotations_by_layer[layer_name].append(annot.rect)

# Print annotations grouped by layer
for layer, annotations in annotations_by_layer.items():
    print(f"Layer: {layer}, Annotations: {annotations}")
```



This script groups annotations by their associated layer and prints their bounding boxes.

---

## 🧠 Tips for Working with Layers in AutoCAD PDFs

* **Layer Visibility**: Use `doc.layer_ui_configs()` to inspect and manage layer visibility.
* **Layer-Specific Extraction**: Toggle layers on or off using `doc.set_layer_ui_config()` to extract content specific to a layer.
* **Annotations**: Annotations can be associated with specific layers. Use `annot.get_oc()` to retrieve the OCG (Optional Content Group) associated with an annotation.
* **Performance**: Be mindful of performance when dealing with large PDFs or numerous layers. Consider processing pages in batches or using asynchronous methods if available.([Artifex][2], [Stack Overflow][3])

---

## 📚 Further Reading

For more detailed information and advanced techniques, refer to the following resources:

* **PyMuPDF Documentation**: Comprehensive guide to PyMuPDF's features and capabilities.
* **Artifex Blog on PDF Layers**: In-depth discussion on accessing and manipulating PDF layers using PyMuPDF.
* **Stack Overflow Discussion**: Community-driven solutions and examples for extracting annotations by layer.([Stack Overflow][3])

If you have specific requirements or encounter any issues while working with your AutoCAD PDFs, feel free to provide more details, and I'll be glad to assist further.

[1]: https://github.com/pymupdf/PyMuPDF/discussions/4091?utm_source=chatgpt.com "Challenges with Extracting and Reapplying Specific Layers from Multi-layer PDFs using PyMuPDF · pymupdf PyMuPDF · Discussion #4091 · GitHub"
[2]: https://artifex.com/blog/optional-content-discovering-the-pdf-layers-pymupdf-python?utm_source=chatgpt.com "Blog - Artifex"
[3]: https://stackoverflow.com/questions/77066094/extract-annotations-by-layer-from-a-pdf-in-python?utm_source=chatgpt.com "Extract annotations by layer from a PDF in Python - Stack Overflow"


==================================================================
==================================================================
==================================================================
Great! Let's walk through a full solution where we:

1. **Render the PDF page as an image** using PyMuPDF.
2. **Use OpenCV to detect rounded rectangles, circles, or ovals**.
3. **Convert the detected image coordinates back to PDF coordinates**.
4. **Extract text from those detected regions** using PyMuPDF.

---

## 🧰 Libraries Required

```bash
pip install pymupdf opencv-python
```

---

## 📷 Step 1: Render PDF Page to Image

```python
import fitz  # PyMuPDF

def render_pdf_page_to_image(pdf_path, page_number=0, zoom=2):
    doc = fitz.open(pdf_path)
    page = doc.load_page(page_number)
    mat = fitz.Matrix(zoom, zoom)  # zoom > 1 = higher resolution
    pix = page.get_pixmap(matrix=mat)
    image_path = f"page_{page_number + 1}.png"
    pix.save(image_path)
    return image_path, zoom
```

---

## 🔍 Step 2: Detect Rounded Rectangles or Circles Using OpenCV

```python
import cv2
import numpy as np

def detect_shapes(image_path):
    img = cv2.imread(image_path)
    gray = cv2.cvtColor(img, cv2.COLOR_BGR2GRAY)
    blurred = cv2.medianBlur(gray, 5)
    edges = cv2.Canny(blurred, 50, 150)

    contours, _ = cv2.findContours(edges, cv2.RETR_EXTERNAL, cv2.CHAIN_APPROX_SIMPLE)

    bounding_boxes = []

    for cnt in contours:
        approx = cv2.approxPolyDP(cnt, 0.02 * cv2.arcLength(cnt, True), True)
        x, y, w, h = cv2.boundingRect(approx)

        # You can add filtering logic here (e.g. min size, aspect ratio)
        if 50 < w < 500 and 50 < h < 500:  # basic filter
            bounding_boxes.append((x, y, x + w, y + h))

    return bounding_boxes
```

---

## 🔁 Step 3: Convert Image Coords Back to PDF Space

PyMuPDF zooms the image, so to get back to **PDF coordinate space**:

```python
def scale_boxes_to_pdf(bounding_boxes, zoom):
    pdf_boxes = []
    for x0, y0, x1, y1 in bounding_boxes:
        pdf_box = (x0 / zoom, y0 / zoom, x1 / zoom, y1 / zoom)
        pdf_boxes.append(pdf_box)
    return pdf_boxes
```

---

## 📄 Step 4: Extract Text Inside Detected Shapes

We use the same rectangular text extraction function from earlier:

```python
def extract_text_in_rects(pdf_path, rect_bboxes, page_number=0):
    doc = fitz.open(pdf_path)
    page = doc.load_page(page_number)
    text_blocks = page.get_text("dict")["blocks"]
    results = []

    for rect in rect_bboxes:
        collected_text = ""
        for block in text_blocks:
            if block["type"] == 0:
                for line in block["lines"]:
                    for span in line["spans"]:
                        mid_x = (span["bbox"][0] + span["bbox"][2]) / 2
                        mid_y = (span["bbox"][1] + span["bbox"][3]) / 2
                        x0, y0, x1, y1 = rect
                        if x0 <= mid_x <= x1 and y0 <= mid_y <= y1:
                            collected_text += span["text"] + " "
        if collected_text:
            results.append(collected_text.strip())
    return results
```

---

## 🧪 Complete Workflow Example

```python
pdf_file = "your_autocad_file.pdf"
page_number = 0

# Step 1: Render page as image
image_path, zoom = render_pdf_page_to_image(pdf_file, page_number)

# Step 2: Detect rounded rectangles or circles
detected_shapes = detect_shapes(image_path)

# Step 3: Convert image boxes to PDF coordinate space
pdf_boxes = scale_boxes_to_pdf(detected_shapes, zoom)

# Step 4: Extract text inside those regions
texts = extract_text_in_rects(pdf_file, pdf_boxes, page_number)

# Output
print("Detected text inside rounded shapes:")
for idx, t in enumerate(texts):
    print(f"[{idx + 1}] {t}")
```

---

## 📌 Notes

* You can tweak the **`detect_shapes`** function to be more aggressive or precise by using techniques like:

  * `cv2.HoughCircles()` for circular shape detection.
  * Checking contour aspect ratio and extent for rounded rectangles.
* For more complex glyphs (e.g. symbols), you can detect shape **and then OCR the region** if needed.

---


==================================================================
==================================================================
==================================================================
PyMuPDF (also known as `fitz`) is a powerful Python library for working with PDF files. It allows you to extract text, images, and metadata, as well as manipulate and analyze the content of PDFs. This is particularly useful when dealing with complex documents like AutoCAD-generated PDFs, which may include vector graphics, annotations, and layered content.([devgem.io][1])

---

## 🔧 What You Can Do with PyMuPDF

### 1. **Extract Text**

You can extract text from PDF pages in various formats:([Artifex][2])

* **Plain Text**: Extracts raw text without any formatting.
* **Blocks**: Returns text as blocks, useful for structured documents.
* **HTML**: Extracts text with basic HTML formatting.
* **JSON**: Provides detailed information about the text layout.([CodeEase][3])

For example, to extract plain text:

```python
import fitz  # PyMuPDF

doc = fitz.open("your_file.pdf")
for page in doc:
    text = page.get_text("text")
    print(text)
```



This will print the extracted text from each page.

### 2. **Extract Images**

You can extract images embedded in the PDF:

```python
import fitz

doc = fitz.open("your_file.pdf")
for page_num in range(len(doc)):
    page = doc.load_page(page_num)
    image_list = page.get_images(full=True)
    for img_index, img in enumerate(image_list):
        xref = img[0]
        base_image = doc.extract_image(xref)
        image_bytes = base_image["image"]
        with open(f"image{page_num + 1}_{img_index + 1}.png", "wb") as img_file:
            img_file.write(image_bytes)
```



This script saves each image as a PNG file.

### 3. **Extract Metadata**

Retrieve metadata such as author, title, and creation date:([etutorialspoint.com][4])

```python
import fitz

doc = fitz.open("your_file.pdf")
metadata = doc.metadata
print(metadata)
```



This will print the metadata of the PDF.

### 4. **Extract Tables**

PyMuPDF can identify and extract tables from PDFs:([Artifex][2])

```python
import fitz

doc = fitz.open("your_file.pdf")
for page in doc:
    blocks = page.get_text("dict")["blocks"]
    for b in blocks:
        if b["type"] == 0:  # Text block
            print(b["text"])
```



This example prints text blocks, which can be further processed to identify tables.

### 5. **Extract Layout Information**

For more detailed analysis, you can extract layout information:

```python
import fitz

doc = fitz.open("your_file.pdf")
for page in doc:
    blocks = page.get_text("dict")["blocks"]
    for b in blocks:
        if b["type"] == 0:  # Text block
            for line in b["lines"]:
                for span in line["spans"]:
                    print(f"Text: {span['text']}, Font: {span['font']}, Size: {span['size']}")
```



This provides detailed information about each text span, including font and size.([Medium][5])

---

## 🧭 PyMuPDF Documentation Highlights

The PyMuPDF tutorial provides comprehensive guidance on:

* **Opening and Navigating PDFs**: Learn how to open PDF files and navigate through pages.
* **Extracting Text and Images**: Detailed methods for extracting text and images from PDFs.
* **Working with Metadata**: Accessing and manipulating PDF metadata.
* **Modifying PDFs**: Techniques for editing and creating PDFs.
* **Advanced Features**: Utilizing PyMuPDF's advanced features for complex PDF manipulation.

For AutoCAD PDFs, which may contain vector graphics and annotations, PyMuPDF's ability to handle complex layouts and extract detailed information makes it a suitable choice. You can use the `get_text("dict")` method to analyze the structure of the document and extract relevant content.([Welcome to python-forum.io][6])

---

## ✅ Sample Workflow for AutoCAD PDFs

Here's a sample workflow to extract and process content from an AutoCAD-generated PDF:

```python
import fitz

doc = fitz.open("your_autocad_file.pdf")
for page_num in range(len(doc)):
    page = doc.load_page(page_num)
    blocks = page.get_text("dict")["blocks"]
    for b in blocks:
        if b["type"] == 0:  # Text block
            print(f"Page {page_num + 1}: {b['text']}")
        elif b["type"] == 1:  # Image block
            xref = b["image"]
            base_image = doc.extract_image(xref)
            image_bytes = base_image["image"]
            with open(f"image{page_num + 1}.png", "wb") as img_file:
                img_file.write(image_bytes)
```



This script processes each page, extracting text and images, and saves the images as PNG files.

---

## 🧪 Tips for Handling AutoCAD PDFs

* **Layered Content**: AutoCAD PDFs may have layered content. Use PyMuPDF's `get_text("dict")` method to analyze and extract specific layers.
* **Vector Graphics**: For vector graphics, PyMuPDF's image extraction capabilities can be utilized, though handling complex vector data may require additional processing.
* **Annotations**: Annotations such as dimensions and notes can be extracted using the `get_text("dict")` method, which provides detailed information about text spans.([Welcome to python-forum.io][6])

---

## 📚 Learn More

For a deeper understanding and more advanced techniques, refer to the [PyMuPDF Tutorial](https://pymupdf.readthedocs.io/en/latest/tutorial.html). It offers detailed explanations and examples to help you effectively work with PDF documents.

If you need assistance

[1]: https://www.devgem.io/posts/extracting-clean-text-from-pdf-using-pymupdf-a-comprehensive-guide?utm_source=chatgpt.com "Extracting Clean Text from PDF using PyMuPDF: A Comprehensive Guide – devgem.io - devgem.io"
[2]: https://artifex.com/blog/table-recognition-extraction-from-pdfs-pymupdf-python?utm_source=chatgpt.com "Table Recognition and Extraction With ..."
[3]: https://codeease.net/programming/python/pymupdf-extract-all-text-from-pdf?utm_source=chatgpt.com "pymupdf extract all text from pdf | Code Ease"
[4]: https://www.etutorialspoint.com/extract-text-from-pdf-using-python?utm_source=chatgpt.com "Extract text from PDF using Python"
[5]: https://neurondai.medium.com/how-to-extract-text-from-a-pdf-using-pymupdf-and-python-caa8487cf9d?utm_source=chatgpt.com "How to Extract Text from a PDF Using PyMuPDF and Python | by Neurond AI | Medium"
[6]: https://python-forum.io/thread-42427.html?utm_source=chatgpt.com "Extract text from PDF"


==================================================================
==================================================================
==================================================================-text-from-pdf?utm_source=chatgpt.com "pymupdf extract all text from pdf | Code Ease"
[4]: https://www.etutorialspoint.com/extract-text-from-pdf-using-python?utm_source=chatgpt.com "Extract text from PDF using Python"
[5]: https://neurondai.medium.com/how-to-extract-text-from-a-pdf-using-pymupdf-and-python-caa8487cf9d?utm_source=chatgpt.com "How to Extract Text from a PDF Using PyMuPDF and Python | by Neurond AI | Medium"
[6]: https://python-forum.io/thread-42427.html?utm_source=chatgpt.com "Extract text from PDF"


==================================================================
==================================================================
==================================================================
