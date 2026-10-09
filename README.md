const MATERIAL_RESISTIVITY = {
  cu: 0.01724,
  copper: 0.01724,
  al: 0.0282,
  aluminum: 0.0282,
  aluminium: 0.0282,
};

const MATERIAL_DENSITY = {
  cu: 1,
  copper: 1,
  al: 0.61,
  aluminum: 0.61,
  aluminium: 0.61,
};

const EPS0 = 8.854187817e-12;
const PI2 = 2 * Math.PI;
const FREQ_HZ = 50;

function parseCSV(text) {
  const rows = []; 
  const lines = text.trim().split(/\r?\n/).filter(line => line.trim() !== "");
  if (lines.length < 2) return rows;

  const headers = parseCSVLine(lines[0]);
  for (let i = 1; i < lines.length; i++) {
    const values = parseCSVLine(lines[i]);
    const record = {};
    headers.forEach((header, index) => {
      record[header.trim()] = values[index] !== undefined ? values[index].trim() : "";
    });
    rows.push(record);
  }
  return rows;
}

function parseCSVLine(line) {
  const result = [];
  let current = "";
  let inQuotes = false;

  for (let i = 0; i < line.length; i++) {
    const ch = line[i];
    if (ch === '"') {
      if (inQuotes && line[i + 1] === '"') {
        current += '"';
        i++;
      } else {
        inQuotes = !inQuotes;
      }
    } else if (ch === ',' && !inQuotes) {
      result.push(current);
      current = "";
    } else {
      current += ch;
    }
  }

  result.push(current);
  return result;
}

function normalizeKey(key) {
  return String(key || "")
    .trim()
    .toLowerCase()
    .replace(/[^a-z0-9]+/g, "_")
    .replace(/^_|_$/g, "");
}

function getValue(row, aliases) {
  const keys = Object.keys(row).map(normalizeKey);
  const normalizedMap = new Map();
  keys.forEach((key, index) => {
    normalizedMap.set(key, Object.values(row)[index]);
  });

  for (const alias of aliases) {
    const key = normalizeKey(alias);
    if (normalizedMap.has(key)) {
      return normalizedMap.get(key);
    }
  }
  return "";
}

function toNumber(value) {
  if (value === undefined || value === null || value === "") return 0;
  const num = String(value).replace(/[^0-9.+\-eE]/g, "");
  const parsed = Number(num);
  return Number.isFinite(parsed) ? parsed : 0;
}

function parseNumericList(raw) {
  if (!raw) return [];
  return String(raw)
    .split(/[;,|]/)
    .map(item => toNumber(item))
    .filter(item => Number.isFinite(item));
}

function parsePositionList(raw) {
  if (!raw) return [];
  const cleaned = String(raw).trim();
  if (!cleaned) return [];

  const tokens = cleaned.split(/[;|]/).map(s => s.trim()).filter(Boolean);
  const positions = [];

  tokens.forEach(token => {
    const pairs = token
      .replace(/[()\[\]]/g, "")
      .split(/\s*,\s*|\s+/)
      .map(v => v.trim())
      .filter(Boolean);

    if (pairs.length >= 2) {
      const x = toNumber(pairs[0]);
      const y = toNumber(pairs[1]);
      if (Number.isFinite(x) && Number.isFinite(y)) {
        positions.push([x, y]);
      }
    }
  });

  return positions;
}

function resolvePhasePositions(row, count = 3) {
  const direct = getValue(row, [
    "phase_positions_m",
    "phase_positions",
    "phase_geometry_m",
    "phase_geometry",
    "positions_phase_m",
    "phases_m",
  ]);

  if (direct) {
    const positions = parsePositionList(direct);
    if (positions.length >= count) return positions.slice(0, count);
  }

  const positions = [];
  for (let i = 0; i < count; i++) {
    const x = getValue(row, [`phase_${i + 1}_x`, `phase_${String.fromCharCode(97 + i)}_x`, `phase_${i + 1}_x_m`, `phase_${String.fromCharCode(97 + i)}_x_m`]);
    const y = getValue(row, [`phase_${i + 1}_y`, `phase_${String.fromCharCode(97 + i)}_y`, `phase_${i + 1}_y_m`, `phase_${String.fromCharCode(97 + i)}_y_m`]);
    const xVal = toNumber(x);
    const yVal = toNumber(y);
    if (Number.isFinite(xVal) && Number.isFinite(yVal)) {
      positions.push([xVal, yVal]);
    }
  }

  if (positions.length === count) return positions;

  const genericX = parseNumericList(getValue(row, ["phase_x_m", "phase_x", "x_positions_m", "x_positions"]));
  const genericY = parseNumericList(getValue(row, ["phase_y_m", "phase_y", "y_positions_m", "y_positions"]));

  if (genericX.length >= count && genericY.length >= count) {
    return genericX.map((x, index) => [x, genericY[index] || 0]);
  }

  return [];
}

function resolveEarthPositions(row) {
  const candidates = [
    "earth_positions_m",
    "earth_positions",
    "earth_geometry_m",
    "earth_geometry",
    "earth_wires_m",
    "ground_wires_m",
    "earth_wire_positions_m",
    "earth_wire_positions",
  ];

  for (const key of candidates) {
    const value = getValue(row, [key]);
    if (value) {
      const positions = parsePositionList(value);
      if (positions.length) return positions;
    }
  }

  const earthPositions = [];
  let idx = 1;
  while (true) {
    const x = getValue(row, [`earth_wire_${idx}_x`, `earth_${idx}_x`, `ground_wire_${idx}_x`, `gw_${idx}_x`]);
    const y = getValue(row, [`earth_wire_${idx}_y`, `earth_${idx}_y`, `ground_wire_${idx}_y`, `gw_${idx}_y`]);
    const xValue = toNumber(x);
    const yValue = toNumber(y);
    if (!Number.isFinite(xValue) || !Number.isFinite(yValue)) break;
    earthPositions.push([xValue, yValue]);
    idx += 1;
  }

  return earthPositions;
}

function resolveDiams(row, count, prefix) {
  const direct = getValue(row, [`${prefix}_diameters_mm`, `${prefix}_diameter_mm`, `${prefix}_diameters`, `${prefix}_diameter`]);
  if (direct) {
    const list = parseNumericList(direct);
    if (list.length >= count) return list.slice(0, count);
  }

  const values = [];
  for (let i = 0; i < count; i++) {
    const candidate = getValue(row, [
      `${prefix}_${i + 1}_diameter_mm`,
      `${prefix}_${String.fromCharCode(97 + i)}_diameter_mm`,
      `${prefix}_${i + 1}_diameter`,
      `${prefix}_${String.fromCharCode(97 + i)}_diameter`,
    ]);
    const value = toNumber(candidate);
    if (value > 0) values.push(value);
  }

  if (values.length >= count) return values;

  const single = toNumber(getValue(row, [`${prefix}_diameter_mm`, `${prefix}_diameter`])) || toNumber(getValue(row, ["conductor_diameter_mm", "wire_diameter_mm", "diameter_mm"]));
  if (single > 0) return Array(count).fill(single);

  return Array(count).fill(0);
}

function resolveConductorArea(row) {
  const direct = toNumber(getValue(row, ["conductor_area_mm2", "area_mm2", "cross_section_mm2", "crosssection_mm2", "area"]));
  if (direct > 0) return direct;

  const diameter = toNumber(getValue(row, ["conductor_diameter_mm", "wire_diameter_mm", "diameter_mm"]));
  if (diameter > 0) {
    const radius = diameter / 2 / 1000;
    return Math.PI * radius * radius * 1e6;
  }

  return 0;
}

function resolveRho(row) {
  const explicit = toNumber(getValue(row, ["conductor_resistivity_ohm_mm2_per_m", "resistivity_ohm_mm2_per_m", "rho_ohm_mm2_per_m", "rho_ohm_m", "resistivity_ohm_m"]));
  if (explicit > 0) return explicit;

  const material = String(getValue(row, ["conductor_material", "material", "phase_material"]) || "").toLowerCase();
  if (MATERIAL_RESISTIVITY[material]) return MATERIAL_RESISTIVITY[material];

  const rperkm = toNumber(getValue(row, ["r_phase_ohm_km", "resistance_ohm_km", "phase_resistance_ohm_km", "resistance_per_km_ohm"]));
  if (rperkm > 0) return rperkm;

  return 0.01724;
}

function geomMeanDistance(points) {
  if (points.length < 2) return 1;
  let product = 1;
  let count = 0;

  for (let i = 0; i < points.length; i++) {
    for (let j = i + 1; j < points.length; j++) {
      const dx = points[i][0] - points[j][0];
      const dy = points[i][1] - points[j][1];
      const distance = Math.sqrt(dx * dx + dy * dy);
      product *= distance;
      count += 1;
    }
  }

  if (count === 0) return 1;
  return Math.pow(product, 1 / count);
}

function geomMeanRadiusForPhase(radiusM) {
  return 0.7788 * radiusM;
}

function phaseResistanceFromRho(areaMm2, rhoOhmMm2PerM) {
  if (!areaMm2 || areaMm2 <= 0) return 0;
  return (rhoOhmMm2PerM * 1000) / areaMm2;
}

function approxPositiveSequenceReactance(phasePositions, radiusM) {
  const gmd = geomMeanDistance(phasePositions);
  const gmr = geomMeanRadiusForPhase(radiusM);
  if (!gmd || !gmr || gmd <= gmr) return 0;
  return 0.1447 * Math.log10(gmd / gmr);
}

function approxZeroSequenceReactance(phasePositions, radiusM, earthPositions) {
  const x1 = approxPositiveSequenceReactance(phasePositions, radiusM);
  const earthInfluence = earthPositions.length > 0 ? 0.12 + 0.08 * earthPositions.length : 0.18;
  return x1 + earthInfluence;
}

function getEquivalentPhaseRadius(diametersMm) {
  const valid = diametersMm.filter(d => d > 0);
  if (!valid.length) return 0;
  const average = valid.reduce((a, b) => a + b, 0) / valid.length;
  return (average / 2) / 1000;
}

function estimateCapacitancePerKm(phasePositions, radiusM) {
  const gmd = geomMeanDistance(phasePositions);
  const r = Math.max(radiusM, 0.0005);
  if (gmd <= r) return 0;
  const lnTerm = Math.log(gmd / r);
  const c = (2 * Math.PI * EPS0) / lnTerm;
  return c * 1000; // F/km
}

function computeOverheadRow(row) {
  const largePhasePositions = resolvePhasePositions(row, 3);
  const earthPositions = resolveEarthPositions(row);
  const phaseDiameters = resolveDiams(row, 3, "phase");
  const earthDiameters = resolveDiams(row, earthPositions.length || 1, "earth");

  const area = resolveConductorArea(row);
  const rho = resolveRho(row);
  const phaseRadius = getEquivalentPhaseRadius(phaseDiameters) || (Math.sqrt(area / Math.PI) * 1000 / 1000);

  const r1 = phaseResistanceFromRho(area || 1, rho);
  const x1 = approxPositiveSequenceReactance(largePhasePositions.length ? largePhasePositions : [[0, 0], [7, 0], [14, 0]], phaseRadius);
  const x0 = approxZeroSequenceReactance(largePhasePositions.length ? largePhasePositions : [[0, 0], [7, 0], [14, 0]], phaseRadius, earthPositions);

  const c1 = estimateCapacitancePerKm(largePhasePositions.length ? largePhasePositions : [[0, 0], [7, 0], [14, 0]], phaseRadius);
  const b1 = PI2 * FREQ_HZ * c1;
  const b0 = Math.max(b1 * 1.7, b1);

  const r0 = r1 * (1 + (earthPositions.length ? 0.18 * earthPositions.length : 0.1));

  return {
    R1: r1,
    R0: r0,
    X1: x1,
    X0: x0,
    B1: b1,
    B0: b0,
    source: "overhead",
  };
}

function resolveCableType(row) {
  const value = String(getValue(row, ["cable_type", "cable_kind", "type", "construction"]) || "").toLowerCase();
  if (value.includes("be")) return "BE";
  if (value.includes("cb")) return "CB";
  if (value.includes("spb")) return "SPB";
  if (value.includes("belt")) return "BE";
  if (value.includes("concentric")) return "CB";
  return "SPB";
}

function resolveReturnWireRadius(row, returnAreaMm2) {
  const diameter = toNumber(getValue(row, ["return_wire_diameter_mm", "ecc_diameter_mm", "earth_return_diameter_mm", "return_diameter_mm"]));
  if (diameter > 0) return diameter / 2 / 1000;
  if (returnAreaMm2 > 0) return Math.sqrt(returnAreaMm2 / Math.PI) / 1000;
  return 0.0015;
}

function computeCableRow(row) {
  const phasePositions = resolvePhasePositions(row, 3);
  const eccPositions = parsePositionList(getValue(row, ["ecc_positions_m", "ecc_positions", "ecc_geometry_m", "ecc_geometry", "return_positions_m", "return_positions"])) || [];
  const cableType = resolveCableType(row);
  const conductorMaterial = String(getValue(row, ["conductor_material", "material", "phase_material"]) || "").toLowerCase();
  const area = resolveConductorArea(row);
  const rho = MATERIAL_RESISTIVITY[conductorMaterial] || resolveRho(row);
  const returnArea = toNumber(getValue(row, ["return_wire_area_mm2", "ecc_area_mm2", "earth_return_area_mm2", "return_area_mm2"]));
  const returnMaterial = String(getValue(row, ["return_wire_material", "ecc_material", "return_material"]) || "").toLowerCase();
  const returnRho = MATERIAL_RESISTIVITY[returnMaterial] || rho;

  const phaseRadius = toNumber(getValue(row, ["phase_diameter_mm", "conductor_diameter_mm", "wire_diameter_mm", "diameter_mm"])) / 2 / 1000 || Math.sqrt(area / Math.PI) / 1000;
  const gmd = geomMeanDistance(phasePositions.length ? phasePositions : [[0, 0], [0.12, 0], [0.24, 0]]);
  const gmr = geomMeanRadiusForPhase(phaseRadius);

  const r1 = phaseResistanceFromRho(area || 1, rho);
  const r0 = r1 + (returnArea > 0 ? ((returnRho * 1000) / returnArea) * 1.8 : 0.12 * r1);

  const l1 = 2e-7 * Math.log(gmd / gmr);
  const x1 = PI2 * FREQ_HZ * l1 * 1000;

  const sheathFactor = cableType === "BE" ? 1.35 : cableType === "CB" ? 1.2 : 1.1;
  const x0 = x1 * sheathFactor;

  const c1 = (2 * Math.PI * EPS0) / Math.log(gmd / Math.max(phaseRadius, 0.0005));
  const b1 = PI2 * FREQ_HZ * c1 * 1000;
  const b0 = b1 * (cableType === "BE" ? 1.8 : cableType === "CB" ? 1.6 : 1.4);

  return {
    R1: r1,
    R0: r0,
    X1: x1,
    X0: x0,
    B1: b1,
    B0: b0,
    source: "cable",
  };
}

function calculateRows(type, dataRows) {
  return dataRows.map(row => {
    const result = type === "overhead" ? computeOverheadRow(row) : computeCableRow(row);
    return { ...row, ...result };
  });
}

function buildTable(rows) {
  const table = document.querySelector("#resultTable");
  const thead = table.querySelector("thead");
  const tbody = table.querySelector("tbody");

  if (!rows.length) {
    thead.innerHTML = "";
    tbody.innerHTML = "<tr><td colspan='10'>No data to display. Upload a CSV file.</td></tr>";
    return;
  }

  const allKeys = Object.keys(rows[0]);
  const keys = [
    ...allKeys.filter(key => !["source"].includes(key)),
    "R1",
    "R0",
    "X1",
    "X0",
    "B1",
    "B0",
  ].filter((key, index, arr) => arr.indexOf(key) === index);

  const headerRow = `<tr>${keys.map(key => `<th>${key}</th>`).join("")}</tr>`;
  thead.innerHTML = headerRow;

  tbody.innerHTML = rows.map((row) => {
    return `<tr>${keys.map(key => `<td>${formatCellValue(row[key])}</td>`).join("")}</tr>`;
  }).join("");
}

function formatCellValue(value) {
  if (value === undefined || value === null || value === "") return "";
  const num = Number(value);
  if (Number.isFinite(num)) {
    return Number.isInteger(num) ? String(num) : num.toFixed(4);
  }
  return String(value);
}

function generateTemplate(type) {
  if (type === "overhead") {
    return [
      "line_name,phase_positions_m,earth_positions_m,conductor_material,conductor_area_mm2,conductor_diameter_mm,conductor_resistivity_ohm_mm2_per_m,phase_diameters_mm,earth_diameters_mm",
      "Line A,0,0; 7,0; 14,0,3.5,8; 10.5,8,Cu,500,12.5,0.01724,12.5,12.5",
      "Line B,0,0; 8,0; 16,0,4,8; 12,8,Al,630,14.3,0.0282,14.3,14.3",
    ].join("\n");
  }

  return [
    "line_name,cable_type,conductor_material,conductor_area_mm2,conductor_diameter_mm,phase_positions_m,return_wire_area_mm2,return_wire_material,ecc_positions_m",
    "Cable A,BE,Cu,500,12.5,0,0; 0.12,0; 0.24,0,160,Cu,0.08,0; 0.18,0",
    "Cable B,SPB,Al,800,18,0,0; 0.12,0; 0.24,0,250,Al,0.06,0; 0.18,0",
  ].join("\n");
}

function triggerDownload(filename, content) {
  const blob = new Blob([content], { type: "text/csv;charset=utf-8;" });
  const url = URL.createObjectURL(blob);
  const a = document.createElement("a");
  a.href = url;
  a.download = filename;
  a.click();
  URL.revokeObjectURL(url);
}

let currentData = [];

function onCalculate() {
  const fileInput = document.getElementById("csvFile");
  const type = document.getElementById("lineType").value;

  if (!fileInput.files || !fileInput.files[0]) {
    alert("Please choose a CSV file first.");
    return;
  }

  const reader = new FileReader();
  reader.onload = function (event) {
    const text = event.target.result;
    const rows = parseCSV(text);
    if (!rows.length) {
      alert("No records found in the CSV file.");
      return;
    }

    currentData = calculateRows(type, rows);
    buildTable(currentData);
  };
  reader.readAsText(fileInput.files[0]);
}

function onDownloadTemplate() {
  const type = document.getElementById("lineType").value;
  triggerDownload(`${type}-template.csv`, generateTemplate(type));
}

function onDownloadSample() {
  const type = document.getElementById("lineType").value;
  triggerDownload(`${type}-sample.csv`, generateTemplate(type));
}

function onDownloadResultCsv() {
  if (!currentData.length) {
    alert("There are no calculated results to export.");
    return;
  }

  const headers = Object.keys(currentData[0]);
  const rows = currentData.map(row => headers.map(key => `"${String(row[key] ?? "").replace(/"/g, '""')}"`).join(","));
  const csv = [headers.join(","), ...rows].join("\n");
  triggerDownload(`hv-results-${document.getElementById("lineType").value}.csv`, csv);
}

document.getElementById("calculateBtn").addEventListener("click", onCalculate);
document.getElementById("templateBtn").addEventListener("click", onDownloadTemplate);
document.getElementById("sampleBtn").addEventListener("click", onDownloadSample);
document.getElementById("downloadCsvBtn").addEventListener("click", onDownloadResultCsv);

const initialTemplate = generateTemplate("overhead");
currentData = calculateRows("overhead", parseCSV(initialTemplate));
buildTable(currentData);
