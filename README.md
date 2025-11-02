# pip-install--r-requirement.
import os
import fitz  # PyMuPDF
from flask import Flask, render_template_string, request
from transformers import pipeline
from werkzeug.utils import secure_filename

app = Flask(__name__)
app.config["UPLOAD_FOLDER"] = "uploads"

# اطمینان از وجود پوشه آپلود
os.makedirs(app.config["UPLOAD_FOLDER"], exist_ok=True)

# مدل خلاصه‌سازی از Hugging Face
summarizer = pipeline("summarization", model="facebook/bart-large-cnn")

HTML = """
<!DOCTYPE html>
<html>
<head>
    <title>📚 PDF Summarizer</title>
    <style>
        body { font-family: Arial; background: #0f172a; color: #f8fafc; text-align: center; padding: 40px; }
        .box { background: #1e293b; padding: 20px; border-radius: 10px; margin-top: 20px; }
        input[type=file] { padding: 10px; background: #1e293b; color: white; border-radius: 5px; border: none; }
        button { padding: 10px 20px; background: #2563eb; border: none; color: white; border-radius: 5px; cursor: pointer; }
        button:hover { background: #1d4ed8; }
    </style>
</head>
<body>
    <h1>📄 PDF Text Summarizer</h1>
    <form method="POST" enctype="multipart/form-data">
        <input type="file" name="pdf" accept=".pdf" required><br><br>
        <button type="submit">Upload & Summarize</button>
    </form>

    {% if summary %}
        <div class="box">
            <h3>🧠 Summary:</h3>
            <p>{{ summary }}</p>
        </div>
    {% endif %}
</body>
</html>
"""

def extract_text_from_pdf(filepath):
    text = ""
    doc = fitz.open(filepath)
    for page in doc:
        text += page.get_text()
    doc.close()
    return text

@app.route("/", methods=["GET", "POST"])
def index():
    summary = None
    if request.method == "POST":
        file = request.files["pdf"]
        if file:
            filename = secure_filename(file.filename)
            filepath = os.path.join(app.config["UPLOAD_FOLDER"], filename)
            file.save(filepath)

            pdf_text = extract_text_from_pdf(filepath)
            if len(pdf_text.strip()) < 100:
                summary = "The PDF is too short to summarize."
            else:
                result = summarizer(pdf_text[:2000], max_length=150, min_length=50, do_sample=False)
                summary = result[0]["summary_text"]
    return render_template_string(HTML, summary=summary)

if __name__ == "__main__":
    app.run(debug=True)
