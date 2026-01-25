/* * Excalidraw Script: Gantt Matrix (Precision Fix)
* Description: 
* - Fixes 'Dec 25' visual bug.
* - Ensures Jan 01 aligns exactly with the first grid line.
*/

// --- CONFIGURATION ---
const settings = {
    cleanCanvas: false, 

    // FOLDERS
    objFolder: "020_objetivo",
    resFolder: "030_resultado", 
    entFolder: "040_entrega",

    // TIMELINE LOCK
    timelineStart: "2026-01-01", 
    timelineEnd:   "2026-12-31", 
    
    pixelsPerDay: 5,      
    gridColor: "2f9e44", 

    // LAYOUT
    startX: 0,
    startY: 10000,          
    colWidth: 250,        
    gapX: 20,             
    timelineStartX: 600,  

    // ROW SIZING
    padding: 10,          
    minHeight: 50,        
    gapY: 10,             

    // STYLE
    fontSize: 16,
    fontFamily: 2,        
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

// Get bounds from SETTINGS (Strict)
function getGridBounds() {
    // Normalize to Midnight to avoid timezone offsets causing 1-day drift
    const min = new Date(settings.timelineStart + "T00:00:00");
    const max = new Date(settings.timelineEnd + "T00:00:00");
    return { min, max };
}

function getXFromDate(dateObj, minDate) {
    // Force UTC math to prevent Daylight Savings Time shifting pixels
    const diffTime = dateObj.getTime() - minDate.getTime();
    const diffDays = Math.floor(diffTime / (1000 * 60 * 60 * 24));
    
    // Add 1px buffer so it doesn't overlap the border exactly
    return settings.timelineStartX + (diffDays * settings.pixelsPerDay) + 1; 
}

// --- MAIN LOGIC ---

const fObj = app.vault.getAbstractFileByPath(settings.objFolder);
const fRes = app.vault.getAbstractFileByPath(settings.resFolder);
const fEnt = app.vault.getAbstractFileByPath(settings.entFolder);

if (!fObj || !fRes || !fEnt) { new Notice("❌ Error: Check folder paths."); return; }

const objFiles = fObj.children.filter(f => f.hasOwnProperty('extension'));
const resFiles = fRes.children.filter(f => f.hasOwnProperty('extension'));
const entFiles = fEnt.children.filter(f => f.hasOwnProperty('extension'));

const { min: globalStart, max: globalEnd } = getGridBounds();

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
            // Safe Date Parsing (Append T00:00:00 to prevent timezone drift)
            let start = entData.startDate ? new Date(entData.startDate + "T00:00:00") : new Date(globalStart);
            let end = entData.dueDate ? new Date(entData.dueDate + "T00:00:00") : new Date(start);
            
            if (isNaN(start) || start < globalStart) start = new Date(globalStart);
            if (isNaN(end) || end < start) { end = new Date(start); end.setDate(end.getDate() + 5); }

            const xPos = getXFromDate(start, globalStart);
            const wPx = Math.max(50, getXFromDate(end, globalStart) - xPos); 

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

// A. Draw Grid 
ea.style.strokeColor = "#000000";
ea.style.fontFamily = settings.fontFamily;
ea.style.fontSize = settings.fontSize;

let loopDate = new Date(globalStart);
// No setDate(1) here - assume GlobalStart (Jan 1) is correct
const gridBottomY = settings.startY + totalChartHeight;

while (loopDate <= globalEnd) {
    const x = getXFromDate(loopDate, globalStart);
    
    // Grid Line
    ea.style.strokeColor = settings.gridColor;
    ea.style.strokeWidth = 1;
    // Draw the line at X - 1 to encapsulate the day
    ea.addLine([[x-1, settings.startY - 30], [x-1, gridBottomY]]);

    // Month Label (Shifted Right by +15px to center in column)
    const monthName = loopDate.toLocaleString('default', { month: 'short', year: '2-digit' });
    ea.style.strokeColor = "#888888"; 
    ea.addText(x + 10, settings.startY - 50, monthName);

    // Increment Month safely
    // We go to the first day of next month to avoid "30th Feb" issues
    loopDate.setDate(1); 
    loopDate.setMonth(loopDate.getMonth() + 1);
}

// B. Draw Objects
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
new Notice(`✅ Fixed Alignment 2026`);