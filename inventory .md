inventory project
from flask import Flask, render_template_string, request, jsonify, Response
import sqlite3
import csv
import io
from datetime import datetime

app = Flask(__name__)
DB_NAME = "factory_inventory.db"


def init_db():
    conn = sqlite3.connect(DB_NAME)
    cursor = conn.cursor()

    # 1. Raw Materials Table
    cursor.execute("""
        CREATE TABLE IF NOT EXISTS materials (
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            material_type TEXT NOT NULL,
            category_color TEXT NOT NULL,
            quantity REAL NOT NULL,
            unit TEXT NOT NULL
        )
    """)

    # 2. Finished Production Count Table
    cursor.execute("""
        CREATE TABLE IF NOT EXISTS production_summary (
            category TEXT PRIMARY KEY,
            total_pairs INTEGER NOT NULL
        )
    """)

    # 3. Production History Log Table
    cursor.execute("""
        CREATE TABLE IF NOT EXISTS production_history (
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            timestamp TEXT NOT NULL,
            category TEXT NOT NULL,
            pairs_made INTEGER NOT NULL,
            pu_leather_used TEXT,
            soles_used TEXT,
            buckles_used TEXT,
            ties_used TEXT,
            heels_used TEXT
        )
    """)

    # Seed Default Data if Tables are Empty
    cursor.execute("SELECT COUNT(*) FROM materials")
    if cursor.fetchone()[0] == 0:
        default_materials = [
            ("PU Leather", "Black", 300.0, "Meters"),
            ("PU Leather", "Brown", 250.0, "Meters"),
            ("PU Leather", "Tan", 200.0, "Meters"),
            ("PU Leather", "Maroon", 150.0, "Meters"),
            ("PU Leather", "Golden", 100.0, "Meters"),
            ("Rubber Sole", "For Shoes", 200.0, "Pairs"),
            ("Rubber Sole", "For Slippers", 300.0, "Pairs"),
            ("Buckles", "Gold", 500.0, "Pieces"),
            ("Buckles", "Silver", 400.0, "Pieces"),
            ("Buckles", "Antique Brass", 250.0, "Pieces"),
            ("Buckles", "Black", 300.0, "Pairs"),
            ("Ties / Laces", "Black", 300.0, "Pairs"),
            ("Ties / Laces", "Brown", 250.0, "Pairs"),
            ("Ties / Laces", "Beige", 200.0, "Pairs"),
            ("Ties / Laces", "Red", 150.0, "Pairs"),
            ("Heels (Shoes)", "1.5 inch", 120.0, "Pairs"),
            ("Heels (Shoes)", "2.0 inch", 150.0, "Pairs"),
            ("Heels (Shoes)", "3.0 inch", 100.0, "Pairs"),
            ("Heels (Shoes)", "4.0 inch", 80.0, "Pairs"),
            ("Heels (Slippers)", "0.5 inch (Flat)", 200.0, "Pairs"),
            ("Heels (Slippers)", "1.0 inch", 180.0, "Pairs"),
            ("Heels (Slippers)", "2.0 inch", 120.0, "Pairs"),
        ]
        cursor.executemany("INSERT INTO materials (material_type, category_color, quantity, unit) VALUES (?, ?, ?, ?)", default_materials)

    cursor.execute("SELECT COUNT(*) FROM production_summary")
    if cursor.fetchone()[0] == 0:
        cursor.execute("INSERT INTO production_summary (category, total_pairs) VALUES ('Shoes', 0)")
        cursor.execute("INSERT INTO production_summary (category, total_pairs) VALUES ('Slippers', 0)")

    conn.commit()
    conn.close()


def get_db_connection():
    conn = sqlite3.connect(DB_NAME)
    conn.row_factory = sqlite3.Row
    return conn


# =========================================================================
# RESPONSIVE FRONTEND HTML / BOOTSTRAP 5 TEMPLATE
# =========================================================================
HTML_TEMPLATE = """
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Pakistani Women's Footwear Factory - SQLite Inventory System</title>
    <!-- Bootstrap 5 & Icons -->
    <link href="https://cdn.jsdelivr.net/npm/bootstrap@5.3.2/dist/css/bootstrap.min.css" rel="stylesheet">
    <link href="https://cdn.jsdelivr.net/npm/bootstrap-icons@1.11.1/font/bootstrap-icons.css" rel="stylesheet">
    <style>
        body { background-color: #f1f5f9; font-family: 'Segoe UI', system-ui, sans-serif; }
        .navbar { background: linear-gradient(135deg, #0f172a 0%, #1e293b 100%); border-bottom: 3px solid #f59e0b; }
        .stat-card { border: none; border-radius: 14px; transition: transform 0.2s; }
        .stat-card:hover { transform: translateY(-3px); }
        .nav-pills .nav-link { color: #334155; font-weight: 600; border-radius: 10px; padding: 10px 18px; margin-right: 6px; }
        .nav-pills .nav-link.active { background-color: #0f172a; color: #ffffff; }
        .card-panel { border: none; border-radius: 16px; box-shadow: 0 4px 12px rgba(0,0,0,0.05); }
        .badge-low { background-color: #fee2e2; color: #991b1b; font-weight: 600; }
        .badge-ok { background-color: #dcfce7; color: #166534; font-weight: 600; }
        .receipt-box { background-color: #0f172a; color: #38bdf8; font-family: 'Consolas', monospace; border-radius: 12px; padding: 16px; white-space: pre-wrap; }
    </style>
</head>
<body>

<nav class="navbar navbar-dark py-3 shadow-sm">
    <div class="container">
        <span class="navbar-brand mb-0 h1">
            <i class="bi bi-shop me-2 text-warning"></i>Pakistani Women's Shoe & Slipper Factory (SQLite DB)
        </span>
        <span class="badge bg-warning text-dark px-3 py-2">Connected to Database</span>
    </div>
</nav>

<div class="container my-4">
    <!-- KPI Summary Cards -->
    <div class="row g-3 mb-4">
        <div class="col-12 col-md-4">
            <div class="card stat-card bg-primary text-white p-3 shadow-sm">
                <div class="d-flex justify-content-between align-items-center">
                    <div>
                        <h6 class="text-uppercase mb-1 opacity-75">Total Shoes Produced</h6>
                        <h2 class="fw-bold mb-0" id="kpi-shoes">0 Pairs</h2>
                    </div>
                    <i class="bi bi-bag-heart-fill fs-1 opacity-50"></i>
                </div>
            </div>
        </div>
        <div class="col-12 col-md-4">
            <div class="card stat-card text-white p-3 shadow-sm" style="background-color: #be185d;">
                <div class="d-flex justify-content-between align-items-center">
                    <div>
                        <h6 class="text-uppercase mb-1 opacity-75">Total Slippers Produced</h6>
                        <h2 class="fw-bold mb-0" id="kpi-slippers">0 Pairs</h2>
                    </div>
                    <i class="bi bi-stars fs-1 opacity-50"></i>
                </div>
            </div>
        </div>
        <div class="col-12 col-md-4">
            <div class="card stat-card bg-dark text-white p-3 shadow-sm">
                <div class="d-flex justify-content-between align-items-center">
                    <div>
                        <h6 class="text-uppercase mb-1 opacity-75">Low Stock Warnings</h6>
                        <h2 class="fw-bold mb-0 text-warning" id="kpi-alerts">0 Items</h2>
                    </div>
                    <i class="bi bi-exclamation-triangle-fill fs-1 text-warning opacity-75"></i>
                </div>
            </div>
        </div>
    </div>

    <!-- Navigation Tabs -->
    <ul class="nav nav-pills mb-4" id="inventoryTabs" role="tablist">
        <li class="nav-item"><button class="nav-link active" data-bs-toggle="pill" data-bs-target="#tab-dashboard"><i class="bi bi-speedometer2 me-1"></i> 1. Live Stock Dashboard</button></li>
        <li class="nav-item"><button class="nav-link" data-bs-toggle="pill" data-bs-target="#tab-production"><i class="bi bi-hammer me-1"></i> 2. Record Production (Deduct)</button></li>
        <li class="nav-item"><button class="nav-link" data-bs-toggle="pill" data-bs-target="#tab-restock"><i class="bi bi-plus-circle me-1"></i> 3. Restock Materials</button></li>
        <li class="nav-item"><button class="nav-link" data-bs-toggle="pill" data-bs-target="#tab-history"><i class="bi bi-clock-history me-1"></i> 4. Production History & CSV</button></li>
    </ul>

    <div class="tab-content">
        <!-- ================= TAB 1: DASHBOARD ================= -->
        <div class="tab-pane fade show active" id="tab-dashboard">
            <div class="card card-panel p-4">
                <div class="d-flex flex-wrap justify-content-between align-items-center mb-3 gap-2">
                    <h5 class="fw-bold mb-0">Raw Material Inventory (PU Leather, Soles, Buckles, Ties, Heels)</h5>
                    <div class="d-flex gap-2">
                        <input type="text" id="stockSearch" class="form-control form-control-sm" placeholder="Search material or color..." onkeyup="filterStockTable()">
                        <a href="/api/export_stock_csv" class="btn btn-success btn-sm text-nowrap"><i class="bi bi-file-earmark-spreadsheet me-1"></i> Export Stock CSV</a>
                    </div>
                </div>
                <div class="table-responsive">
                    <table class="table table-hover align-middle" id="stockTable">
                        <thead class="table-light">
                            <tr>
                                <th>Material Type</th>
                                <th>Color / Category / Heel Height</th>
                                <th class="text-center">Available Stock</th>
                                <th class="text-center">Unit</th>
                                <th class="text-center">Status</th>
                            </tr>
                        </thead>
                        <tbody id="stockTableBody"></tbody>
                    </table>
                </div>
            </div>
        </div>

        <!-- ================= TAB 2: PRODUCTION ================= -->
        <div class="tab-pane fade" id="tab-production">
            <div class="card card-panel p-4">
                <h5 class="fw-bold mb-3">Manufacture Shoes or Slippers & Deduct Used Materials from DB</h5>
                <form id="productionForm" onsubmit="submitProduction(event)">
                    <div class="row g-3">
                        <div class="col-12 col-md-6">
                            <label class="form-label fw-semibold">1. Footwear Category</label>
                            <select id="prod_category" class="form-select" onchange="updateHeelDropdown()">
                                <option value="Shoes">Shoes</option>
                                <option value="Slippers">Slippers (Chappal / Sandals)</option>
                            </select>
                        </div>
                        <div class="col-12 col-md-6">
                            <label class="form-label fw-semibold">Pairs Produced</label>
                            <input type="number" id="prod_pairs" class="form-control" value="10" min="1" required>
                        </div>

                        <div class="col-12 col-md-6">
                            <label class="form-label fw-semibold">2. PU Leather Color</label>
                            <select id="prod_pu_color" class="form-select"></select>
                        </div>
                        <div class="col-12 col-md-6">
                            <label class="form-label fw-semibold">Meters Used</label>
                            <input type="number" step="0.01" id="prod_pu_used" class="form-control" value="5.5" min="0" required>
                        </div>

                        <div class="col-12 col-md-6">
                            <label class="form-label fw-semibold">3. Rubber Soles Used (Pairs)</label>
                            <input type="number" id="prod_soles_used" class="form-control" value="10" min="0" required>
                        </div>
                        <div class="col-12 col-md-6"></div>

                        <div class="col-12 col-md-6">
                            <label class="form-label fw-semibold">4. Buckle Color</label>
                            <select id="prod_buckle_color" class="form-select"></select>
                        </div>
                        <div class="col-12 col-md-6">
                            <label class="form-label fw-semibold">Buckle Pieces Used</label>
                            <input type="number" id="prod_buckles_used" class="form-control" value="0" min="0" required>
                        </div>

                        <div class="col-12 col-md-6">
                            <label class="form-label fw-semibold">5. Tie / Lace Color</label>
                            <select id="prod_tie_color" class="form-select"></select>
                        </div>
                        <div class="col-12 col-md-6">
                            <label class="form-label fw-semibold">Tie Pairs Used</label>
                            <input type="number" id="prod_ties_used" class="form-control" value="0" min="0" required>
                        </div>

                        <div class="col-12 col-md-6">
                            <label class="form-label fw-semibold">6. Heel Height (Inches)</label>
                            <select id="prod_heel_size" class="form-select"></select>
                        </div>
                        <div class="col-12 col-md-6">
                            <label class="form-label fw-semibold">Heel Pairs Used</label>
                            <input type="number" id="prod_heels_used" class="form-control" value="10" min="0" required>
                        </div>

                        <div class="col-12 mt-4">
                            <button type="submit" class="btn btn-success btn-lg w-100 fw-bold"><i class="bi bi-check2-circle me-1"></i> Record Production & Deduct Inventory</button>
                        </div>
                    </div>
                </form>

                <div class="mt-4">
                    <label class="form-label fw-bold">Remaining Stock Deduction Receipt:</label>
                    <div class="receipt-box" id="receiptOutput">Record a production batch above to view remaining material quantities...</div>
                </div>
            </div>
        </div>

        <!-- ================= TAB 3: RESTOCK ================= -->
        <div class="tab-pane fade" id="tab-restock">
            <div class="card card-panel p-4">
                <h5 class="fw-bold mb-3">Add New Stock or Register New Colors & Heel Sizes</h5>
                <form id="restockForm" onsubmit="submitRestock(event)">
                    <div class="row g-3">
                        <div class="col-12 col-md-6">
                            <label class="form-label fw-semibold">Select Material Type</label>
                            <select id="restock_type" class="form-select" onchange="updateRestockFields()">
                                <option value="PU Leather">PU Leather (Meters)</option>
                                <option value="Rubber Sole">Rubber Soles (Shoes / Slippers)</option>
                                <option value="Buckles">Buckles (By Color)</option>
                                <option value="Ties / Laces">Ties (By Color)</option>
                                <option value="Heels">Heels (By Category & Inches)</option>
                            </select>
                        </div>

                        <div class="col-12 col-md-6" id="restock_cat_wrapper" style="display:none;">
                            <label class="form-label fw-semibold">Footwear Category</label>
                            <select id="restock_cat" class="form-select">
                                <option value="For Shoes">Shoes</option>
                                <option value="For Slippers">Slippers</option>
                            </select>
                        </div>

                        <div class="col-12 col-md-6" id="restock_attr_wrapper">
                            <label class="form-label fw-semibold" id="restock_attr_label">Color Name (e.g., Black, Golden)</label>
                            <input type="text" id="restock_attr" class="form-control" value="Black">
                        </div>

                        <div class="col-12 col-md-6">
                            <label class="form-label fw-semibold">Quantity to Add</label>
                            <input type="number" step="0.01" id="restock_qty" class="form-control" value="100" min="0.01" required>
                        </div>

                        <div class="col-12 mt-4">
                            <button type="submit" class="btn btn-primary btn-lg w-100 fw-bold"><i class="bi bi-box-arrow-in-down me-1"></i> Add Stock to Database</button>
                        </div>
                    </div>
                </form>
            </div>
        </div>

        <!-- ================= TAB 4: HISTORY ================= -->
        <div class="tab-pane fade" id="tab-history">
            <div class="card card-panel p-4">
                <div class="d-flex flex-wrap justify-content-between align-items-center mb-3 gap-2">
                    <h5 class="fw-bold mb-0">Batch Production History Log</h5>
                    <div class="d-flex gap-2">
                        <a href="/api/export_history_csv" class="btn btn-success btn-sm"><i class="bi bi-download me-1"></i> Export History CSV</a>
                        <button class="btn btn-danger btn-sm" onclick="clearHistory()"><i class="bi bi-trash me-1"></i> Clear Log</button>
                    </div>
                </div>
                <div class="table-responsive">
                    <table class="table table-striped table-hover align-middle">
                        <thead class="table-dark">
                            <tr>
                                <th>Date & Time</th>
                                <th>Category</th>
                                <th>Pairs Made</th>
                                <th>PU Leather Used</th>
                                <th>Soles Used</th>
                                <th>Buckles Used</th>
                                <th>Ties Used</th>
                                <th>Heels Used</th>
                            </tr>
                        </thead>
                        <tbody id="historyTableBody"></tbody>
                    </table>
                </div>
            </div>
        </div>
    </div>
</div>

<script src="https://cdn.jsdelivr.net/npm/bootstrap@5.3.2/dist/js/bootstrap.bundle.min.js"></script>
<script>
    let inventoryData = {};

    async function fetchInventory() {
        const res = await fetch('/api/data');
        inventoryData = await res.json();
        renderDashboard();
    }

    function renderDashboard() {
        // KPI Cards
        document.getElementById('kpi-shoes').innerText = `${inventoryData.summary.Shoes} Pairs`;
        document.getElementById('kpi-slippers').innerText = `${inventoryData.summary.Slippers} Pairs`;

        // Stock Table
        const tbody = document.getElementById('stockTableBody');
        tbody.innerHTML = '';
        let lowCount = 0;

        inventoryData.materials.forEach(m => {
            const isLow = m.quantity <= (m.material_type.includes('PU') ? 30 : 25);
            if (isLow) lowCount++;
            const badge = isLow ? `<span class="badge badge-low px-3 py-2">LOW STOCK</span>` : `<span class="badge badge-ok px-3 py-2">Available</span>`;
            
            tbody.innerHTML += `
                <tr>
                    <td class="fw-semibold">${m.material_type}</td>
                    <td>${m.category_color}</td>
                    <td class="text-center fw-bold">${m.quantity.toFixed(2)}</td>
                    <td class="text-center text-muted">${m.unit}</td>
                    <td class="text-center">${badge}</td>
                </tr>`;
        });
        document.getElementById('kpi-alerts').innerText = `${lowCount} Items`;

        // Populate Production Dropdowns
        const puColors = inventoryData.materials.filter(m => m.material_type === 'PU Leather').map(m => m.category_color);
        const buckleColors = inventoryData.materials.filter(m => m.material_type === 'Buckles').map(m => m.category_color);
        const tieColors = inventoryData.materials.filter(m => m.material_type === 'Ties / Laces').map(m => m.category_color);

        populateSelect('prod_pu_color', puColors);
        populateSelect('prod_buckle_color', ['None', ...buckleColors]);
        populateSelect('prod_tie_color', ['None', ...tieColors]);
        updateHeelOptions();

        // Populate History
        const hbody = document.getElementById('historyTableBody');
        hbody.innerHTML = '';
        if (inventoryData.history.length === 0) {
            hbody.innerHTML = `<tr><td colspan="8" class="text-center text-muted py-4">No production batches recorded yet.</td></tr>`;
        } else {
            inventoryData.history.forEach(item => {
                hbody.innerHTML += `
                    <tr>
                        <td class="text-nowrap">${item.timestamp}</td>
                        <td><span class="badge bg-secondary">${item.category}</span></td>
                        <td class="fw-bold">${item.pairs_made}</td>
                        <td>${item.pu_leather_used || 'None'}</td>
                        <td>${item.soles_used || 'None'}</td>
                        <td>${item.buckles_used || 'None'}</td>
                        <td>${item.ties_used || 'None'}</td>
                        <td>${item.heels_used || 'None'}</td>
                    </tr>`;
            });
        }
    }

    function populateSelect(id, items) {
        const el = document.getElementById(id);
        const prev = el.value;
        el.innerHTML = items.map(o => `<option value="${o}">${o}</option>`).join('');
        if (items.includes(prev)) el.value = prev;
    }

    function updateHeelOptions() {
        const cat = document.getElementById('prod_category').value;
        const heelMatType = cat === 'Shoes' ? 'Heels (Shoes)' : 'Heels (Slippers)';
        const sizes = inventoryData.materials.filter(m => m.material_type === heelMatType).map(m => m.category_color);
        populateSelect('prod_heel_size', ['None', ...sizes]);
    }

    function updateRestockFields() {
        const type = document.getElementById('restock_type').value;
        const catWrapper = document.getElementById('restock_cat_wrapper');
        const attrWrapper = document.getElementById('restock_attr_wrapper');
        const label = document.getElementById('restock_attr_label');

        if (type === 'PU Leather') {
            catWrapper.style.display = 'none';
            attrWrapper.style.display = 'block';
            label.innerText = 'PU Leather Color (e.g., Black, Tan):';
        } else if (type === 'Rubber Sole') {
            catWrapper.style.display = 'block';
            attrWrapper.style.display = 'none';
        } else if (type === 'Buckles' || type === 'Ties / Laces') {
            catWrapper.style.display = 'none';
            attrWrapper.style.display = 'block';
            label.innerText = 'Color Name (e.g., Gold, Black):';
        } else if (type === 'Heels') {
            catWrapper.style.display = 'block';
            attrWrapper.style.display = 'block';
            label.innerText = 'Heel Height in Inches (e.g., 2.0 inch):';
        }
    }

    async function submitProduction(e) {
        e.preventDefault();
        const payload = {
            category: document.getElementById('prod_category').value,
            pairs_made: parseInt(document.getElementById('prod_pairs').value),
            pu_color: document.getElementById('prod_pu_color').value,
            pu_used: parseFloat(document.getElementById('prod_pu_used').value),
            soles_used: parseInt(document.getElementById('prod_soles_used').value),
            buckle_color: document.getElementById('prod_buckle_color').value,
            buckles_used: parseInt(document.getElementById('prod_buckles_used').value),
            tie_color: document.getElementById('prod_tie_color').value,
            ties_used: parseInt(document.getElementById('prod_ties_used').value),
            heel_size: document.getElementById('prod_heel_size').value,
            heels_used: parseInt(document.getElementById('prod_heels_used').value)
        };

        const res = await fetch('/api/produce', {
            method: 'POST',
            headers: { 'Content-Type': 'application/json' },
            body: JSON.stringify(payload)
        });
        const data = await res.json();
        if (!res.ok) {
            alert(data.error || 'Error recording production');
            return;
        }
        document.getElementById('receiptOutput').innerText = data.receipt;
        fetchInventory();
    }

    async function submitRestock(e) {
        e.preventDefault();
        const type = document.getElementById('restock_type').value;
        let matType = type;
        if (type === 'Heels') {
            const cat = document.getElementById('restock_cat').value;
            matType = cat === 'For Shoes' ? 'Heels (Shoes)' : 'Heels (Slippers)';
        }

        const payload = {
            material_type: matType,
            category_color: type === 'Rubber Sole' ? document.getElementById('restock_cat').value : document.getElementById('restock_attr').value.trim(),
            quantity: parseFloat(document.getElementById('restock_qty').value)
        };

        const res = await fetch('/api/restock', {
            method: 'POST',
            headers: { 'Content-Type': 'application/json' },
            body: JSON.stringify(payload)
        });
        const data = await res.json();
        if (!res.ok) {
            alert(data.error || 'Error restocking');
            return;
        }
        alert(data.message);
        fetchInventory();
    }

    async function clearHistory() {
        if (!confirm('Clear production history log?')) return;
        await fetch('/api/clear_history', { method: 'POST' });
        fetchInventory();
    }

    function filterStockTable() {
        const query = document.getElementById('stockSearch').value.toLowerCase();
        document.querySelectorAll('#stockTableBody tr').forEach(row => {
            row.style.display = row.innerText.toLowerCase().includes(query) ? '' : 'none';
        });
    }

    window.onload = fetchInventory;
</script>
</body>
</html>
"""


# =========================================================================
# FLASK BACKEND API & SQLITE HANDLERS
# =========================================================================
@app.route("/")
def index():
    return render_template_string(HTML_TEMPLATE)


@app.route("/api/data", methods=["GET"])
def get_data():
    conn = get_db_connection()
    materials = conn.execute("SELECT * FROM materials ORDER BY material_type, category_color").fetchall()
    summary_rows = conn.execute("SELECT * FROM production_summary").fetchall()
    history = conn.execute("SELECT * FROM production_history ORDER BY id DESC").fetchall()
    conn.close()

    summary = {row["category"]: row["total_pairs"] for row in summary_rows}
    return jsonify({
        "materials": [dict(m) for m in materials],
        "summary": summary,
        "history": [dict(h) for h in history]
    })


@app.route("/api/produce", methods=["POST"])
def produce():
    data = request.json
    cat = data["category"]
    pairs = int(data["pairs_made"])

    pu_color = data["pu_color"]
    pu_used = float(data["pu_used"])
    soles_used = int(data["soles_used"])
    sole_cat = f"For {cat}"

    buckle_color = data["buckle_color"]
    buckles_used = int(data["buckles_used"]) if buckle_color != "None" else 0

    tie_color = data["tie_color"]
    ties_used = int(data["ties_used"]) if tie_color != "None" else 0

    heel_size = data["heel_size"]
    heels_used = int(data["heels_used"]) if heel_size != "None" else 0
    heel_mat_type = f"Heels ({cat})"

    conn = get_db_connection()
    cursor = conn.cursor()

    # Check Stock Levels in SQLite
    def check_and_get(m_type, c_color, required):
        row = cursor.execute("SELECT quantity FROM materials WHERE material_type = ? AND category_color = ?", (m_type, c_color)).fetchone()
        if not row or row["quantity"] < required:
            raise ValueError(f"Insufficient stock for {c_color} ({m_type})! Available: {row['quantity'] if row else 0}")
        return row["quantity"]

    try:
        check_and_get("PU Leather", pu_color, pu_used)
        check_and_get("Rubber Sole", sole_cat, soles_used)
        if buckle_color != "None" and buckles_used > 0:
            check_and_get("Buckles", buckle_color, buckles_used)
        if tie_color != "None" and ties_used > 0:
            check_and_get("Ties / Laces", tie_color, ties_used)
        if heel_size != "None" and heels_used > 0:
            check_and_get(heel_mat_type, heel_size, heels_used)

        # Deduct from Database
        cursor.execute("UPDATE materials SET quantity = quantity - ? WHERE material_type = 'PU Leather' AND category_color = ?", (pu_used, pu_color))
        cursor.execute("UPDATE materials SET quantity = quantity - ? WHERE material_type = 'Rubber Sole' AND category_color = ?", (soles_used, sole_cat))
        if buckle_color != "None" and buckles_used > 0:
            cursor.execute("UPDATE materials SET quantity = quantity - ? WHERE material_type = 'Buckles' AND category_color = ?", (buckles_used, buckle_color))
        if tie_color != "None" and ties_used > 0:
            cursor.execute("UPDATE materials SET quantity = quantity - ? WHERE material_type = 'Ties / Laces' AND category_color = ?", (ties_used, tie_color))
        if heel_size != "None" and heels_used > 0:
            cursor.execute("UPDATE materials SET quantity = quantity - ? WHERE material_type = ? AND category_color = ?", (heels_used, heel_mat_type, heel_size))

        # Update Finished Goods Total
        cursor.execute("UPDATE production_summary SET total_pairs = total_pairs + ? WHERE category = ?", (pairs, cat))

        # Get remaining balances for receipt
        rem_pu = cursor.execute("SELECT quantity FROM materials WHERE material_type = 'PU Leather' AND category_color = ?", (pu_color,)).fetchone()["quantity"]
        rem_soles = cursor.execute("SELECT quantity FROM materials WHERE material_type = 'Rubber Sole' AND category_color = ?", (sole_cat,)).fetchone()["quantity"]

        timestamp = datetime.now().strftime("%Y-%m-%d %H:%M:%S")
        cursor.execute("""
            INSERT INTO production_history (timestamp, category, pairs_made, pu_leather_used, soles_used, buckles_used, ties_used, heels_used)
            VALUES (?, ?, ?, ?, ?, ?, ?, ?)
        """, (
            timestamp, cat, pairs,
            f"{pu_color} ({pu_used}m)",
            f"{soles_used} pairs",
            f"{buckle_color} ({buckles_used} pcs)" if buckle_color != "None" else None,
            f"{tie_color} ({ties_used} pairs)" if tie_color != "None" else None,
            f"{heel_size} ({heels_used} pairs)" if heel_size != "None" else None
        ))

        conn.commit()
    except ValueError as e:
        conn.close()
        return jsonify({"error": str(e)}), 400

    conn.close()

    receipt = (
        f"[{timestamp}] SUCCESS: +{pairs} Pairs of {cat} Produced!\n"
        f"----------------------------------------------------\n"
        f"• {pu_color} PU Leather Remaining : {rem_pu:.2f} Meters\n"
        f"• {cat} Rubber Soles Remaining: {rem_soles} Pairs\n"
    )
    return jsonify({"receipt": receipt})


@app.route("/api/restock", methods=["POST"])
def restock():
    data = request.json
    m_type = data["material_type"]
    c_color = data["category_color"].strip().title() if data["material_type"] != "Rubber Sole" else data["category_color"]
    qty = float(data["quantity"])

    conn = get_db_connection()
    cursor = conn.cursor()

    row = cursor.execute("SELECT id FROM materials WHERE material_type = ? AND category_color = ?", (m_type, c_color)).fetchone()
    if row:
        cursor.execute("UPDATE materials SET quantity = quantity + ? WHERE id = ?", (qty, row["id"]))
    else:
        unit = "Pairs" if "Heels" in m_type or m_type == "Rubber Sole" or m_type == "Ties / Laces" else ("Pieces" if m_type == "Buckles" else "Meters")
        cursor.execute("INSERT INTO materials (material_type, category_color, quantity, unit) VALUES (?, ?, ?, ?)", (m_type, c_color, qty, unit))

    conn.commit()
    conn.close()
    return jsonify({"message": f"Successfully added {qty} to {c_color} ({m_type})!"})


@app.route("/api/clear_history", methods=["POST"])
def clear_history():
    conn = get_db_connection()
    conn.execute("DELETE FROM production_history")
    conn.commit()
    conn.close()
    return jsonify({"status": "cleared"})


@app.route("/api/export_history_csv")
def export_history():
    conn = get_db_connection()
    history = conn.execute("SELECT * FROM production_history").fetchall()
    conn.close()

    output = io.StringIO()
    writer = csv.writer(output)
    writer.writerow(["ID", "Timestamp", "Category", "Pairs Made", "PU Leather Used", "Soles Used", "Buckles Used", "Ties Used", "Heels Used"])
    for h in history:
        writer.writerow([h["id"], h["timestamp"], h["category"], h["pairs_made"], h["pu_leather_used"], h["soles_used"], h["buckles_used"], h["ties_used"], h["heels_used"]])

    return Response(output.getvalue(), mimetype="text/csv", headers={"Content-Disposition": "attachment;filename=production_history.csv"})


@app.route("/api/export_stock_csv")
def export_stock():
    conn = get_db_connection()
    materials = conn.execute("SELECT * FROM materials").fetchall()
    conn.close()

    output = io.StringIO()
    writer = csv.writer(output)
    writer.writerow(["Material Type", "Color / Category", "Quantity", "Unit"])
    for m in materials:
        writer.writerow([m["material_type"], m["category_color"], m["quantity"], m["unit"]])

    return Response(output.getvalue(), mimetype="text/csv", headers={"Content-Disposition": "attachment;filename=current_stock.csv"})


if __name__ == "__main__":
    init_db()
    print("Starting SQLite Factory Web Server at http://127.0.0.1:5000")
    app.run(host="0.0.0.0", port=5000, debug=True)
