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
