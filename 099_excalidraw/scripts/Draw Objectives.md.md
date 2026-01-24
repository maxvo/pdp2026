/* * Excalidraw Script: Objective Matrix + Gantt Timeline (FIXED)
* Description: 
* - Corrected 'ea.addLine' syntax error.
* - Generates timeline grid and places deliverables by date.
*/

// --- CONFIGURATION ---
const settings = {
    cleanCanvas: true, 

    // FOLDERS
    objFolder: "020_objetivo",
    resFolder: "030_resultado", 
    entFolder: "040_entrega",

    // LAYOUT (Left Side)
    startX: 0,
    startY: 100,          // Moved down for headers
    colWidth: 250,        
    gapX: 20,             
    
    // TIMELINE (Right Side)
    timelineStartX: 600,  
    pixelsPerDay: 2,      // Scale (increase to stretch timeline)
    gridColor: "#e9ecef", 

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

// --- DATE MATH ---
function getGlobalDateRange(files) {
    let min = new Date(); 
    let max = new Date();
    max.setMonth(max.getMonth() + 3); 

    let found = false;
    for (const f of files) {
        const c = app.metadataCache.getFileCache(f);
        if (c?.frontmatter?.startDate) {
            const d = new Date(c.frontmatter.startDate);
            if (!isNaN(d) && d < min) min = d;
            found = true;
        }
        if (c?.frontmatter?.dueDate) {
            const d = new Date(c.frontmatter.dueDate);
            if (!isNaN(d) && d > max) max = d;
            found = true;
        }
    }
    // Add 15 days buffer
    min.setDate(min.getDate() - 15);
    max.setDate(max.getDate() + 15);
    return { min, max };
}

function getXFromDate(dateObj, minDate) {
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

const { min: globalStart, max: globalEnd } = getGlobalDateRange(entFiles);

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

// Reset
ea.reset();
if (settings.cleanCanvas) ea.clear();

// --- DRAW TIMELINE GRID ---
ea.style.strokeColor = "#000000";
ea.style.fontFamily = settings.fontFamily;
ea.style.fontSize = settings.fontSize;

let loopDate = new Date(globalStart);
loopDate.setDate(1); // Snap to 1st

// Calculate approximate total height for grid lines
const estimatedTotalHeight = Math.max(500, objFiles.length * 300); 

while (loopDate <= globalEnd) {
    const x = getXFromDate(loopDate, globalStart);
    
    // 1. Draw Grid Line (FIXED SYNTAX)
    ea.style.strokeColor = settings.gridColor;
    ea.style.strokeWidth = 1;
    
    // Note the double brackets [[x,y], [x,y]]
    ea.addLine([
        [x, settings.startY - 30], 
        [x, settings.startY + estimatedTotalHeight]
    ]);

    // 2. Draw Month Label
    const monthName = loopDate.toLocaleString('default', { month: 'short', year: '2-digit' });
    ea.style.strokeColor = "#888888"; 
    ea.addText(x + 5, settings.startY - 50, monthName);

    loopDate.setMonth(loopDate.getMonth() + 1);
}

// Reset styles for content
ea.style.backgroundColor = "transparent";
ea.style.fillStyle = "hachure";
ea.style.strokeWidth = 1;
ea.style.textAlign = "left";
ea.style.verticalAlign = "top";

let currentY = settings.startY;
const maxTextWidth = settings.colWidth - (settings.padding * 2);

// --- DRAW ROWS ---
for (let i = 0; i < objFiles.length; i++) {
    const objFile = objFiles[i];
    const myResults = mapResToObj[objFile.basename] || [];
    const rowData = []; 
    let totalRowHeight = 0;

    for (const resFile of myResults) {
        const myEntregas = mapEntToRes[resFile.basename] || [];
        
        const entLayouts = [];
        let entStackHeight = 0;

        for (const entData of myEntregas) {
            let start = entData.startDate ? new Date(entData.startDate) : new Date(globalStart);
            let end = entData.dueDate ? new Date(entData.dueDate) : new Date(start);
            
            if (isNaN(start)) start = new Date(globalStart);
            if (isNaN(end) || end < start) { 
                end = new Date(start); 
                end.setDate(end.getDate() + 5);
            }

            const xPos = getXFromDate(start, globalStart);
            const wPx = Math.max(50, getXFromDate(end, globalStart) - xPos); 

            const textData = getTextHeight(entData.file.basename, wPx - (settings.padding*2));
            const hPx = Math.max(settings.minHeight, textData.height);

            entLayouts.push({
                x: xPos,
                width: wPx,
                height: hPx,
                text: textData.text
            });
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
        totalRowHeight += resFinalHeight;
    }
    if (rowData.length > 0) totalRowHeight += (rowData.length - 1) * settings.gapY;

    const objTextData = getTextHeight(objFile.basename, maxTextWidth);
    const objFinalHeight = Math.max(settings.minHeight, objTextData.height, totalRowHeight);

    // Draw
    const color = settings.colors[i % settings.colors.length];
    ea.style.strokeColor = color;

    // Col 1
    const objRectId = ea.addRect(settings.startX, currentY, settings.colWidth, objFinalHeight);
    const objTextId = ea.addText(settings.startX + settings.padding, currentY + settings.padding, objTextData.text);
    ea.addToGroup([objRectId, objTextId]);

    // Col 2
    let resY = currentY;
    const resX = settings.startX + settings.colWidth + settings.gapX;

    for (const resItem of rowData) {
        const resRectId = ea.addRect(resX, resY, settings.colWidth, resItem.resHeight);
        const resTextId = ea.addText(resX + settings.padding, resY + settings.padding, resItem.resText);
        ea.addToGroup([resRectId, resTextId]);

        // Col 3 (Timeline)
        let entY = resY;
        for (const entItem of resItem.entregas) {
            const entRectId = ea.addRect(entItem.x, entY, entItem.width, entItem.height);
            const entTextId = ea.addText(entItem.x + settings.padding, entY + settings.padding, entItem.text);
            ea.addToGroup([entRectId, entTextId]);
            entY += entItem.height + settings.gapY;
        }
        resY += resItem.resHeight + settings.gapY;
    }

    currentY += objFinalHeight + settings.gapY;
}

await ea.addElementsToView(true, true, true);
new Notice(`✅ Gantt Timeline Generated.`);