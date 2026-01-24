/* * Excalidraw Script: 3-Level Matrix (Production Fixed)
* Description: 
* - Configurable 'Clean Canvas' to prevent pile-ups.
* - Debug logs for positioning.
* - 3 Columns: Objective -> Result -> Entrega (Time-based width)
*/

// --- CONFIGURATION ---
const settings = {
    // 1. CLEAR CANVAS? (Set to true to wipe drawing before running)
    cleanCanvas: true, 

    // 2. FOLDERS
    objFolder: "020_objetivo",
    resFolder: "030_resultado", 
    entFolder: "040_entrega",

    // 3. STARTING POSITION (Check if these match what you want!)
    startX: 0,
    startY: 0,

    // 4. DIMENSIONS
    colWidth: 300,        
    gapX: 50,             // Horizontal Gap
    gapY: 10,             // Vertical Gap
    padding: 10,          
    minHeight: 60,        

    // 5. ENTREGA TIME SCALING
    baseWidth: 100,       
    pixelsPerDay: 5,      
    defaultDuration: 5,   

    // 6. STYLE
    fontSize: 20,
    fontFamily: 1,        
    colors: ["#4a6fa5", "#5c8a8a"] 
};

// --- HELPERS ---
function cleanLink(linkValue) {
    if (!linkValue) return "";
    let raw = Array.isArray(linkValue) ? linkValue[0] : linkValue;
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
    return {
        text: wrapped,
        height: metrics.height + (settings.padding * 2)
    };
}

function getEntregaDimensions(itemData) {
    let days = settings.defaultDuration;
    if (itemData.startDate && itemData.dueDate) {
        const start = new Date(itemData.startDate);
        const end = new Date(itemData.dueDate);
        if (!isNaN(start.getTime()) && !isNaN(end.getTime())) {
            const diffTime = Math.abs(end - start);
            days = Math.ceil(diffTime / (1000 * 60 * 60 * 24)); 
            if (days < 1) days = 1; 
        }
    }
    const calculatedWidth = settings.baseWidth + (days * settings.pixelsPerDay);
    const textData = getTextHeight(itemData.file.basename, calculatedWidth - (settings.padding * 2));
    const finalHeight = Math.max(settings.minHeight, textData.height);

    return {
        width: calculatedWidth,
        height: finalHeight,
        text: textData.text
    };
}

// --- MAIN LOGIC ---

// 1. Files
const fObj = app.vault.getAbstractFileByPath(settings.objFolder);
const fRes = app.vault.getAbstractFileByPath(settings.resFolder);
const fEnt = app.vault.getAbstractFileByPath(settings.entFolder);

if (!fObj || !fRes || !fEnt) {
    new Notice("❌ Error: Folders not found. Check settings.");
    return;
}

const objFiles = fObj.children.filter(f => f.hasOwnProperty('extension'));
const resFiles = fRes.children.filter(f => f.hasOwnProperty('extension'));
const entFiles = fEnt.children.filter(f => f.hasOwnProperty('extension'));

// 2. Maps
const mapResToObj = {}; 
const mapEntToRes = {}; 
objFiles.forEach(f => mapResToObj[f.basename] = []);
resFiles.forEach(f => mapEntToRes[f.basename] = []);

for (const ent of entFiles) {
    const cache = app.metadataCache.getFileCache(ent);
    if (cache?.frontmatter?.resultado) {
        const targetRes = cleanLink(cache.frontmatter.resultado);
        if (mapEntToRes[targetRes]) mapEntToRes[targetRes].push({ file: ent, ...cache.frontmatter });
    }
}
for (const res of resFiles) {
    const cache = app.metadataCache.getFileCache(res);
    if (cache?.frontmatter?.objetivo) {
        const targetObj = cleanLink(cache.frontmatter.objetivo);
        if (mapResToObj[targetObj]) mapResToObj[targetObj].push(res);
    }
}

// 3. Setup Canvas
ea.reset();

// -> CLEAN CANVAS OPTION
if (settings.cleanCanvas) {
    ea.clear(); 
}

ea.style.backgroundColor = "transparent";
ea.style.fillStyle = "hachure";
ea.style.fontFamily = settings.fontFamily;
ea.style.fontSize = settings.fontSize;
ea.style.textAlign = "left";
ea.style.verticalAlign = "top";
ea.style.strokeWidth = 1;

let currentY = settings.startY;
const maxTextWidth = settings.colWidth - (settings.padding * 2);

console.log("--- Starting Draw Loop ---");

// 4. Loop
for (let i = 0; i < objFiles.length; i++) {
    const objFile = objFiles[i];
    const myResults = mapResToObj[objFile.basename] || [];
    const rowData = []; 
    let totalRowHeight = 0;

    // A. Calculate Heights
    for (const resFile of myResults) {
        const myEntregas = mapEntToRes[resFile.basename] || [];
        let entStackHeight = 0;
        const entLayouts = [];
        
        for (const entData of myEntregas) {
            const dims = getEntregaDimensions(entData);
            entLayouts.push(dims);
            entStackHeight += dims.height;
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

    // DEBUG: Print Position
    console.log(`Row ${i}: Y=${currentY}, Height=${objFinalHeight}`);

    // B. Draw
    const color = settings.colors[i % settings.colors.length];
    ea.style.strokeColor = color;

    // Draw Objective
    const objRectId = ea.addRect(settings.startX, currentY, settings.colWidth, objFinalHeight);
    const objTextId = ea.addText(settings.startX + settings.padding, currentY + settings.padding, objTextData.text);
    ea.addToGroup([objRectId, objTextId]);

    // Draw Results & Entregas
    let resY = currentY;
    const resX = settings.startX + settings.colWidth + settings.gapX;
    const entX = resX + settings.colWidth + settings.gapX;

    for (const resItem of rowData) {
        // Result
        const resRectId = ea.addRect(resX, resY, settings.colWidth, resItem.resHeight);
        const resTextId = ea.addText(resX + settings.padding, resY + settings.padding, resItem.resText);
        ea.addToGroup([resRectId, resTextId]);

        // Entregas
        let entY = resY;
        for (const entItem of resItem.entregas) {
            const entRectId = ea.addRect(entX, entY, entItem.width, entItem.height);
            const entTextId = ea.addText(entX + settings.padding, entY + settings.padding, entItem.text);
            ea.addToGroup([entRectId, entTextId]);
            entY += entItem.height + settings.gapY;
        }
        resY += resItem.resHeight + settings.gapY;
    }

    currentY += objFinalHeight + settings.gapY;
}

await ea.addElementsToView(true, true, true);
new Notice(`✅ Drawn ${objFiles.length} Rows. Check Console for Y-coords.`);