import io, re, json, tempfile, os
from flask import Flask, render_template, request, send_file
import pdfplumber, pandas as pd

app = Flask(__name__)

# ── simple pattern: 20x16  •  16X14  •  Ø 8"
DUCT_RE = re.compile(r'\b(\d{1,2})[xX](\d{1,2})\b|\bØ?\s?(\d{1,2})["”]')

def extract_rows(pdf_bytes: bytes):
    rows = []
    with pdfplumber.open(io.BytesIO(pdf_bytes)) as pdf:
        for page in pdf.pages:
            for line in (page.extract_text() or "").splitlines():
                m = DUCT_RE.search(line)
                if m:
                    w, h, d = m.groups()
                    rows.append({
                        "Qty": 1,
                        "Shape": "Round" if d else "Rect",
                        "Width_in":  w or "",
                        "Height_in": h or "",
                        "Diameter_in": d or "",
                        "Length_ft": "",
                        "CID": 866 if d else 2,
                        "Gauge": "24GA_G90",
                        "Notes": "auto‑parsed"
                    })
    return rows

@app.route("/", methods=["GET", "POST"])
def home():
    if request.method == "POST":
        pdf = request.files["pdf"]
        rows = extract_rows(pdf.read())
        return render_template("index.html", rows=json.dumps(rows))
    return render_template("index.html", rows="[]")

@app.route("/export", methods=["POST"])
def export_csv():
    rows = json.loads(request.form["rows"])
    df = pd.DataFrame(rows)
    tmp = tempfile.NamedTemporaryFile(delete=False, suffix=".csv")
    df.to_csv(tmp.name, index=False)
    return send_file(tmp.name, as_attachment=True,
                     download_name="FabShop_List.csv",
                     mimetype="text/csv")

if __name__ == "__main__":
    port = int(os.environ.get("PORT", 8000))
    app.run(host="0.0.0.0", port=port)
flask==2.3.3
pdfplumber==0.10.3
pandas==2.2.2
<!doctype html>
<html lang="en" x-data="{rows: {{ rows|safe }}}">
<head>
  <script src="https://cdn.tailwindcss.com"></script>
  <script src="https://cdn.jsdelivr.net/npm/alpinejs@3.x.x/dist/cdn.min.js" defer></script>
  <meta charset="utf-8">
  <title>Duct‑Take‑Off</title>
</head>
<body class="p-6">
  <h1 class="text-2xl font-bold mb-4">Duct‑Take‑Off Calculator</h1>

  <!-- PDF upload -->
  <form x-show="rows.length==0" method="post" enctype="multipart/form-data"
        class="border p-4 rounded">
    <input type="file" name="pdf" accept="application/pdf" required>
    <button class="ml-4 px-3 py-1 bg-blue-600 text-white rounded">Upload</button>
  </form>

  <!-- editable grid -->
  <template x-if="rows.length">
    <form action="/export" method="post" class="mt-4">
      <input type="hidden" name="rows" :value="JSON.stringify(rows)">
      <table class="border w-full text-sm">
        <thead class="bg-gray-200">
          <tr><th>Qty</th><th>Shape</th><th>W</th><th>H</th><th>Ø</th><th>Length (ft)</th></tr>
        </thead>
        <tbody>
          <template x-for="(r,i) in rows" :key="i">
            <tr>
              <td><input x-model.number="r.Qty"        class="w-12 border"></td>
              <td x-text="r.Shape"></td>
              <td><input x-model="r.Width_in"          class="w-12 border"></td>
              <td><input x-model="r.Height_in"         class="w-12 border"></td>
              <td><input x-model="r.Diameter_in"       class="w-12 border"></td>
              <td><input x-model="r.Length_ft"         class="w-16 border"></td>
            </tr>
          </template>
        </tbody>
      </table>
      <button class="mt-4 px-3 py-1 bg-green-600 text-white rounded">
        Export FabShop CSV
      </button>
    </form>
  </template>
</body>
</html>
venv/
__pycache__/
*.pyc
.DS_Store
*.pdf
