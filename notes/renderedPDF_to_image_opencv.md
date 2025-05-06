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
