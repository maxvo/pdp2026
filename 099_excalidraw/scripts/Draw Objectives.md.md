/* * Excalidraw Script: Strategic Matrix (Obj > Res > Entrega)
* Description: Generates a 3-level hierarchy. 
* - Handles Results with missing Entregas (draws Result only).
* - Auto-scales heights based on content.
* - Auto-scales Entrega width based on duration.
*/

// --- CONFIGURATION ---
const settings = {
    // 1. Folder Paths (Case Sensitive!)
    objFolder: "020_objetivo",
    resFolder: "030_resultado", 
    entFolder: "040_entrega",

    // 2. Layout Dimensions
    colWidth: 300,        // Width of Obj and Res columns
    gapX: 50,             // Horizontal space between columns
    gapY: 10,             // Vertical space between blocks
    padding: 10,          // Text padding
    minHeight: 60,        // Minimum height for any block
    
    // 3. Time Visualization (Entrega Width)
    baseWidth: 100,       // Minimum width (for 1 day or missing dates)
    pixelsPerDay: 5,      // Width added per day
    defaultDuration: 5,   // Fallback if dates are missing

    // 4. Style
    fontSize: 20,
    fontFamily: 1,        // 1:Hand, 2:Normal, 3:Code
    colors: ["#4a6fa5", "#5c8a8a"] // Alternating row colors
};

// --- HELPER FUNCTIONS ---

// Cleans [[WikiLinks]] or returns raw text
function cleanLink(linkValue) {
    if (!linkValue) return "";
    let raw = Array.isArray(linkValue) ? linkValue[0] : linkValue;
    if (typeof raw !== 'string') return String(raw) || "";
    return raw.replace(/\[\[|\]\]/g, "").split("|")[0];
}

// Wraps text to fit inside the box
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

// Calculates text height
function getTextHeight(text, maxWidth) {
    const wrapped = wrapText(text, maxWidth);
    const metrics = ea.measureText(wrapped);
    return {
        text: wrapped,
        height: metrics.height + (settings.padding * 2)
    };
}

// Calculates width/height for Deliverables based on dates
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

// --- MAIN EXECUTION ---

// 1. Validate Folders
const fObj = app.vault.getAbstractFileByPath(settings.objFolder);
const fRes = app.vault.getAbstractFileByPath(settings.resFolder);
const fEnt = app.vault.getAbstractFileByPath(settings.entFolder);

if (!fObj || !fRes || !fEnt) {
    new Notice("❌ Error: One or more folders not found. Check settings.");
    return;
}

const objFiles = fObj.children.filter(f => f.hasOwnProperty('extension'));
const resFiles = fRes.children.filter(f => f.hasOwnProperty('extension'));
const entFiles = fEnt.children.filter(f => f.hasOwnProperty('extension'));

// 2. Map Relationships
// We pre-fill maps to ensure even empty results exist in the system
const mapResToObj = {}; 
const mapEntToRes = {}; 

objFiles.forEach(f => mapResToObj[f.basename] = []);
resFiles.forEach(f => mapEntToRes[f.basename] = []);

// Map Deliverables -> Results
for (const ent of entFiles) {
    const cache = app.metadataCache.getFileCache(ent);
    if (cache?.frontmatter?.resultado) {
        const targetRes = cleanLink(cache.frontmatter.resultado);
        if (mapEntToRes[targetRes]) {
            mapEntToRes[targetRes].push({ file: ent, ...cache.frontmatter });
        }
    }
}

// Map Results -> Objectives
for (const res of resFiles) {
    const cache = app.metadataCache.getFileCache(res);
    if (cache?.frontmatter?.objetivo) {
        const targetObj = cleanLink(cache.frontmatter.objetivo);
        if (mapResToObj[targetObj]) {
            mapResToObj[targetObj].push(res);
        }
    }
}

// 3. Initialize Excalidraw
ea.reset();
ea.style.backgroundColor = "transparent";
ea.style.fillStyle = "hachure";
ea.style.fontFamily = settings.fontFamily;
ea.style.fontSize = settings.fontSize;
ea.style.textAlign = "left";
ea.style.verticalAlign = "top";
ea.style.strokeWidth = 1;

let currentY = settings.startY;
const maxTextWidth = settings.colWidth - (settings.padding * 2);

// 4. Iterate Objectives (Rows)
for (let i = 0; i < objFiles.length; i++) {
    const objFile = objFiles[i];
    const myResults = mapResToObj[objFile.basename] || [];
    const rowData = []; 
    let totalRowHeight = 0;

    // --- Calculate Layout (Bottom-Up) ---
    for (const resFile of myResults) {
        const myEntregas = mapEntToRes[resFile.basename] || []; // Returns empty array if no deliveries
        
        let entStackHeight = 0;
        const entLayouts = [];
        
        // Calculate Entregas Stack (Skipped if myEntregas is empty)
        for (const entData of myEntregas) {
            const dims = getEntregaDimensions(entData);
            entLayouts.push(dims);
            entStackHeight += dims.height;
        }
        if (entLayouts.length > 0) entStackHeight += (entLayouts.length - 1) * settings.gapY;

        // Result Height Calculation
        // If entStackHeight is 0, it falls back to minHeight or textHeight.
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
    // Add gaps between Results
    if (rowData.length > 0) totalRowHeight += (rowData.length - 1) * settings.gapY;

    // Objective Height Calculation
    const objTextData = getTextHeight(objFile.basename, maxTextWidth);
    const objFinalHeight = Math.max(settings.minHeight, objTextData.height, totalRowHeight);

    // --- Draw Elements ---
    const color = settings.colors[i % settings.colors.length];
    ea.style.strokeColor = color;

    // Col 1: Objective
    const objRectId = ea.addRect(settings.startX, currentY, settings.colWidth, objFinalHeight);
    const objTextId = ea.addText(settings.startX + settings.padding, currentY + settings.padding, objTextData.text);
    ea.addToGroup([objRectId, objTextId]);

    // Col 2 & 3: Results & Entregas
    let resY = currentY;
    const resX = settings.startX + settings.colWidth + settings.gapX;
    const entX = resX + settings.colWidth + settings.gapX;

    for (const resItem of rowData) {
        // Draw Result
        const resRectId = ea.addRect(resX, resY, settings.colWidth, resItem.resHeight);
        const resTextId = ea.addText(resX + settings.padding, resY + settings.padding, resItem.resText);
        ea.addToGroup([resRectId, resTextId]);

        // Draw Entregas (Only if they exist)
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
new Notice(`✅ Matrix Generated: ${objFiles.length} Objectives.`);