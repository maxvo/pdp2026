/* * Excalidraw Script: Gantt 2026 (Strict Limits)
* Description: 
* - Grid is LOCKED to Jan 2026 - Dec 2026.
* - No extra buffer months.
* - Clickable links.
*/

// --- CONFIGURATION ---
const settings = {
    cleanCanvas: true, 

    // FOLDERS
    objFolder: "020_objetivo",
    resFolder: "030_resultado", 
    entFolder: "040_entrega",

    // TIMELINE LIMITS (STRICT)
    timelineStart: "2026-01-01",
    timelineEnd: "2026-12-31",
    
    pixelsPerDay: 2,      
    gridColor: "#e9ecef", 

    // LAYOUT
    startX: 0,
    startY: 100,          
    colWidth: 250,        
    gapX: 20,             
    timelineStartX: 600,  

    // ROW SIZING
    padding: 10,          
    minHeight: 50,        
    gapY: 10,             

    // STYLE
    fontSize: 16,
    fontFamily: 1,        
    colors: ["#4a6fa5", "#5c8a8a"] 
};

// --- HELPERS ---

function cleanLink(val) {
    if (!val) return "";
    let raw = Array.isArray(val) ? val[0] : val;
    if (typeof raw !== 'string') return String(raw) || "";
    return raw.replace(/\[\[|\]\]/g, "").split("|")[0];
}

function wrapText(text, maxWidth) {
    if (!text) return "";
    const words = text.split(" ");
    let lines = [];
    let currentLine = words[0];
    for (let i = 1; i < words.length; i++) {
        const word = words[i];
        const testLine = currentLine + " " + word;
        if (ea.measureText(testLine).width < maxWidth) {
            currentLine = testLine;
        } else {
            lines.push(currentLine);
            currentLine = word;
        }
    }
    lines.push(currentLine);
    return lines.join("\n");
}

function getTextHeight(text, maxWidth) {
    const wrapped = wrapText(text, maxWidth);
    const metrics = ea.measureText(wrapped);
    return { text: wrapped, height: metrics.height + (settings.padding * 2) };
}

function addLinkedText(x, y, text, filename) {
    const id = ea.addText(x, y, text);
    const el = ea.getElement(id);
    el.link = `[[${filename}]]`; 
    return id;
}

// --- DATE MATH ---

// Calculate X relative to the STRICT START DATE
function getXFromDate(dateObj) {
    const minDate = new Date(settings.timelineStart);
    const diffTime = dateObj - minDate;
    const diffDays = Math.ceil(diffTime / (1000 * 60 * 60 * 24));
    return settings.timelineStartX + (diffDays * settings.pixelsPerDay);
}

// --- MAIN LOGIC ---

const fObj = app.vault.getAbstractFileByPath(settings.objFolder);
const fRes = app.vault.getAbstractFileByPath(settings.resFolder);
const fEnt = app.vault.getAbstractFileByPath(settings.entFolder);

if (!fObj || !fRes || !fEnt) { new Notice("❌ Error: Check folder paths."); return; }

const objFiles = fObj.children.filter(f => f.hasOwnProperty('extension'));
const resFiles = fRes.children.filter(f => f.hasOwnProperty('extension'));
const entFiles = fEnt.children.filter(f => f.hasOwnProperty('extension'));

// Hardcoded Limits
const globalStart = new Date(settings.timelineStart);
const globalEnd = new Date(settings.timelineEnd);

// Map Relationships
const mapResToObj = {}; 
const mapEntToRes = {}; 
objFiles.forEach(f => mapResToObj[f.basename] = []);
resFiles.forEach(f => mapEntToRes[f.basename] = []);

for (const ent of entFiles) {
    const cache = app.metadataCache.getFileCache(ent);
    if (cache?.frontmatter?.resultado) {
        const target = cleanLink(cache.frontmatter.resultado);
        if (mapEntToRes[target]) mapEntToRes[target].push({ file: ent, ...cache.frontmatter });
    }
}
for (const res of resFiles) {
    const cache = app.metadataCache.getFileCache(res);
    if (cache?.frontmatter?.objetivo) {
        const target = cleanLink(cache.frontmatter.objetivo);
        if (mapResToObj[target]) mapResToObj[target].push(res);
    }
}

// --- STEP 1: SIMULATION ---
const maxTextWidth = settings.colWidth - (settings.padding * 2);
let totalChartHeight = 0;
const layoutData = []; 

for (let i = 0; i < objFiles.length; i++) {
    const objFile = objFiles[i];
    const myResults = mapResToObj[objFile.basename] || [];
    const rowData = []; 
    let rowHeightAccumulator = 0;

    for (const resFile of myResults) {
        let myEntregas = mapEntToRes[resFile.basename] || [];
        
        myEntregas.sort((a, b) => {
            const dateA = a.startDate ? new Date(a.startDate).getTime() : 0;
            const dateB = b.startDate ? new Date(b.startDate).getTime() : 0;
            return dateA - dateB;
        });

        const entLayouts = [];
        let entStackHeight = 0;

        for (const entData of myEntregas) {
            let start = entData.startDate ? new Date(entData.startDate) : new Date(globalStart);
            let end = entData.dueDate ? new Date(entData.dueDate) : new Date(start);
            
            // Allow items outside range to exist, but they might be clipped visually
            if (isNaN(start)) start = new Date(globalStart);
            if (isNaN(end) || end < start) { end = new Date(start); end.setDate(end.getDate() + 5); }

            const xPos = getXFromDate(start);
            const wPx = Math.max(50, getXFromDate(end) - xPos); 

            const textData = getTextHeight(entData.file.basename, wPx - (settings.padding*2));
            const hPx = Math.max(settings.minHeight, textData.height);

            entLayouts.push({ x: xPos, width: wPx, height: hPx, text: textData.text, file: entData.file });
            entStackHeight += hPx;
        }
        if (entLayouts.length > 0) entStackHeight += (entLayouts.length - 1) * settings.gapY;

        const resTextData = getTextHeight(resFile.basename, maxTextWidth);
        const resFinalHeight = Math.max(settings.minHeight, resTextData.height, entStackHeight);

        rowData.push({
            resFile: resFile,
            resText: resTextData.text,
            resHeight: resFinalHeight, 
            entregas: entLayouts       
        });
        rowHeightAccumulator += resFinalHeight;
    }
    if (rowData.length > 0) rowHeightAccumulator += (rowData.length - 1) * settings.gapY;

    const objTextData = getTextHeight(objFile.basename, maxTextWidth);
    const objFinalHeight = Math.max(settings.minHeight, objTextData.height, rowHeightAccumulator);

    layoutData.push({
        objFile: objFile,
        objText: objTextData.text,
        objHeight: objFinalHeight,
        rowData: rowData
    });

    totalChartHeight += objFinalHeight + settings.gapY;
}


// --- STEP 2: DRAWING ---

ea.reset();
if (settings.cleanCanvas) ea.clear();

// A. Draw Grid (Locked to 2026)
ea.style.strokeColor = "#000000";
ea.style.fontFamily = settings.fontFamily;
ea.style.fontSize = settings.fontSize;

let loopDate = new Date(globalStart);
// Set loopDate to exactly Jan 1st
loopDate.setDate(1); 
loopDate.setHours(0,0,0,0);

const gridBottomY = settings.startY + totalChartHeight;

// Loop until Dec 31
while (loopDate <= globalEnd) {
    const x = getXFromDate(loopDate);
    
    // Grid Line
    ea.style.strokeColor = settings.gridColor;
    ea.style.strokeWidth = 1;
    ea.addLine([[x, settings.startY - 30], [x, gridBottomY]]);

    // Month Label
    const monthName = loopDate.toLocaleString('default', { month: 'short', year: '2-digit' });
    ea.style.strokeColor = "#888888"; 
    ea.addText(x + 5, settings.startY - 50, monthName);

    // Advance 1 month
    loopDate.setMonth(loopDate.getMonth() + 1);
}

// Reset Styles
ea.style.backgroundColor = "transparent";
ea.style.fillStyle = "hachure";
ea.style.strokeWidth = 1;
ea.style.textAlign = "left";
ea.style.verticalAlign = "top";

let currentY = settings.startY;

for (let i = 0; i < layoutData.length; i++) {
    const row = layoutData[i];
    const color = settings.colors[i % settings.colors.length];
    ea.style.strokeColor = color;

    // 1. Objective
    const objRectId = ea.addRect(settings.startX, currentY, settings.colWidth, row.objHeight);
    const objTextId = addLinkedText(settings.startX + settings.padding, currentY + settings.padding, row.objText, row.objFile.basename);
    ea.addToGroup([objRectId, objTextId]);

    // 2. Results
    let resY = currentY;
    const resX = settings.startX + settings.colWidth + settings.gapX;

    for (const resItem of row.rowData) {
        const resRectId = ea.addRect(resX, resY, settings.colWidth, resItem.resHeight);
        const resTextId = addLinkedText(resX + settings.padding, resY + settings.padding, resItem.resText, resItem.resFile.basename);
        ea.addToGroup([resRectId, resTextId]);

        // 3. Entregas
        let entY = resY;
        for (const entItem of resItem.entregas) {
            const entRectId = ea.addRect(entItem.x, entY, entItem.width, entItem.height);
            const entTextId = addLinkedText(entItem.x + settings.padding, entY + settings.padding, entItem.text, entItem.file.basename);
            ea.addToGroup([entRectId, entTextId]);
            entY += entItem.height + settings.gapY;
        }
        resY += resItem.resHeight + settings.gapY;
    }
    currentY += row.objHeight + settings.gapY;
}

await ea.addElementsToView(true, true, true);
new Notice(`✅ Gantt: 2026 Only.`);