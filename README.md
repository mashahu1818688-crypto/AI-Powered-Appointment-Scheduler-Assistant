


Overview This project is a backend service that parses appointment requests from natural language text or scanned images and converts them into structured appointment data. It integrates OCR, entity extraction, date/time normalization, and exposes a REST API. Features

Accepts appointment requests as plain text or image files (scanned notes, photos).

Uses Tesseract OCR for text extraction from images.

Extracts key entities: department, date phrase, and time phrase.

Normalizes date/time to ISO format in Asia/Kolkata timezone.

Provides clear error responses for ambiguous or incomplete inputs.

Simple REST API built with FastAPI.

Requirements

Python 3.8 or higher

Tesseract OCR (installed on your system)

Linux (Ubuntu/Kali) preferred for easier setup, but should work on Windows/macOS with minor adjustments.

Setup Instructions

    Clone the repository git clone https://github.com/yourusername/appointment-scheduler.git cd appointment-scheduler

    Create and activate a virtual environment python3 -m venv venv source venv/bin/activate # On Windows: venv\Scripts\activate

    Install Python dependencies pip install -r requirements.txt

    Install Tesseract OCR engine

For Ubuntu/Kali:

sudo apt update sudo apt install tesseract-ocr

For Windows:

Download installer from https://github.com/tesseract-ocr/tesseract

    Run the API server uvicorn app.main:app --host 127.0.0.1 --port 8000 --reload

Server will start at: http://127.0.0.1:8000

API Usage POST /parse

Description: Parse appointment request from text or image.

Request Parameters:

Parameter Type Description text string Appointment request in text file file Image file containing request

Response Examples:

Success:

{ "appointment": { "department": "Dentistry", "date": "2026-01-30", "time": "15:00", "tz": "Asia/Kolkata" }, "status": "ok" }

Needs Clarification:

{ "status": "needs_clarification", "message": "Ambiguous date/time or department" }

Testing with Curl

Text input:

curl -X POST "http://127.0.0.1:8000/parse?text=Book%20dentist%20next%20Friday%20at%203pm"

Image input:

curl -X POST -F "file=@/path/to/your/image.jpg" "http://127.0.0.1:8000/parse"

Project Structure appointment-scheduler/ │ ├── app/ │ ├── main.py # FastAPI app and routes │ ├── ocr.py # OCR functions using pytesseract │ ├── normalize.py # Date/time normalization logic │ ├── extract.py # Entity extraction logic │ └── ... │ ├── tests/ # Optional: unit tests ├── requirements.txt # Python dependencies ├── README.md # This file └── ...

Notes

Make sure Tesseract OCR is correctly installed and accessible in your system PATH.

Use python -m pip install --upgrade pip if you run into pip issues.

For ambiguous inputs, the API returns a clarification request—make sure your frontend or client handles this gracefully.

If you want, I can help you with:

Creating a requirements.txt file

Sample code snippets for each module (ocr.py, extract.py, normalize.py, main.py)

Writing the screen recording/demo script checklist

What would you like next?

yes

Awesome! Let’s start step-by-step. First, I’ll help you create the requirements.txt file and then basic code snippets for the 4 main modules:

    requirements.txt fastapi uvicorn python-multipart pytesseract dateparser

    Basic code snippets for your project modules a) app/ocr.py — OCR using pytesseract from PIL import Image import pytesseract from fastapi import UploadFile import io

def extract_text_from_image(file: UploadFile) -> str: image_bytes = file.file.read() image = Image.open(io.BytesIO(image_bytes)) text = pytesseract.image_to_string(image) return text

b) app/extract.py — Extract entities (date, time, department) from text import re

def extract_entities(text: str): # Simple regex based extraction (you can improve this with NLP)

# Extract department (assuming dentist/dentistry, doctor etc.)
department = None
departments = ["dentist", "dentistry", "doctor", "cardiology", "dermatology"]
for dept in departments:
    if dept in text.lower():
        department = dept.capitalize()
        break

# Extract date phrase - naive approach: look for keywords like 'next Friday', 'tomorrow', 'Monday'
date_match = re.search(r'(next \w+|tomorrow|today|monday|tuesday|wednesday|thursday|friday|saturday|sunday)', text, re.IGNORECASE)
date_phrase = date_match.group(0) if date_match else None

# Extract time phrase - naive approach looking for formats like '3pm', '15:00', '3:00 PM'
time_match = re.search(r'(\d{1,2}(:\d{2})?\s?(am|pm)?)', text, re.IGNORECASE)
time_phrase = time_match.group(0) if time_match else None

return {
    "department": department,
    "date_phrase": date_phrase,
    "time_phrase": time_phrase
}

c) app/normalize.py — Normalize date/time to ISO format in Asia/Kolkata timezone import dateparser

def normalize_entities(entities): settings = { "PREFER_DATES_FROM": "future", "TIMEZONE": "Asia/Kolkata", "RETURN_AS_TIMEZONE_AWARE": False }

date = dateparser.parse(entities.get("date_phrase", ""), settings=settings)
time = dateparser.parse(entities.get("time_phrase", ""), settings=settings)

if not date or not time:
    return None, 0.0

return {
    "date": date.date().isoformat(),
    "time": time.strftime("%H:%M"),
    "department": entities["department"]
}, 0.90

d) app/main.py — FastAPI app with /parse endpoint from fastapi import FastAPI, UploadFile, File, Query from app.ocr import extract_text_from_image from app.extract import extract_entities from app.normalize import normalize_entities

app = FastAPI()

@app.post("/parse") async def parse_appointment(text: str = Query(None), file: UploadFile = File(None)): raw_text = "" if file: raw_text = extract_text_from_image(file) elif text: raw_text = text else: return {"status": "error", "message": "No input provided"}

entities = extract_entities(raw_text)
if not all([entities.get("date_phrase"), entities.get("time_phrase"), entities.get("department")]):
    return {"status": "needs_clarification", "message": "Ambiguous date/time or department"}

normalized, confidence = normalize_entities(entities)
if not normalized:
    return {"status": "needs_clarification", "message": "Could not normalize date/time"}

return {
    "appointment": {
        "department": normalized["department"],
        "date": normalized["date"],
        "time": normalized["time"],
        "tz": "Asia/Kolkata"
    },
    "status": "ok"
