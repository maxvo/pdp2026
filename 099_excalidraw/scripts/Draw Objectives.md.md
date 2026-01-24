/* * Excalidraw Script: 3-Level Matrix (Obj -> Res -> Entrega)
* Description: 
* - Column 1: Objectives (Auto-height)
* - Column 2: Results (Auto-height based on Entregas)
* - Column 3: Entregas (Width based on Start/Due Date duration)
*/

// --- CONFIGURATION ---
const settings = {
    // Folders
    objFolder: "020_objetivo",
    resFolder: "030_resultado", // Check if it is singular or plural in your vault
    entFolder: "040_entrega",

    // Layout
    colWidth: 300,        // Standard width for Obj and Res
    gapX: 50,             // Horizontal gap between columns
    gapY: 10,             // Vertical gap between items
    padding: 10,          
    minHeight: 60,        
    
    // Time/Size Configuration for "Entrega"
    baseWidth: 100,       // Minimum width for an Entrega
    pixelsPerDay: 5,      // How much width to add per day of duration
    defaultDuration: 5,   // If dates are missing, assume this many days

    // Font
    fontSize: 20,
    fontFamily: 1, // 1: Hand, 2: Normal, 3: Code

    // Colors (Alternating by Objective Row)
    colors: [
        "#4a6fa5", // Muted Slate Blue
        "#5c8a8a"  // Muted Sage Teal
    ]
};

// --- HELPERS ---

// 1. Clean Link ([[Link]] -> Link)
function cleanLink(linkValue) {
    if (!linkValue) return "";
    let raw = Array.isArray(linkValue) ? linkValue[0] : linkValue;
    if (typeof raw !== 'string') return String(raw) || "";
    return raw.replace(/\[\[|\]\]/g, "").split("|")[0];
}

// 2. Word Wrap
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

// 3. Get Text Height
function getTextHeight(text, maxWidth) {
    const wrapped = wrapText(text, maxWidth);
    const metrics = ea.measureText(wrapped);
    return {
        text: wrapped,
        height: metrics.height + (settings.padding * 2)
    };
}

// 4. Calculate Duration & Width
function getEntregaDimensions(frontmatter) {
    let days = settings.defaultDuration;
    
    if (frontmatter && frontmatter.startDate && frontmatter.dueDate) {
        const start = new Date(frontmatter.startDate);
        const end = new Date(frontmatter.dueDate);
        
        if (!isNaN(start) && !isNaN(end)) {
            // Difference in time / milliseconds in a day
            const diffTime = Math.abs(end - start);
            days = Math.ceil(diffTime / (1000 * 60 * 60 * 24)); 
            if (days < 1) days = 1; // Minimum 1 day
        }
    }

    // Width Calculation: Base + (Days * Multiplier)
    const calculatedWidth = settings.baseWidth + (days * settings.pixelsPerDay);
    
    // We still need to wrap text based on this new width
    const textData = getTextHeight(frontmatter.file.basename, calculatedWidth - (settings.padding * 2));
    const finalHeight = Math.max(settings.minHeight, textData.height);

    return {
        width: calculatedWidth,
        height: finalHeight,
        text: textData.text,
        days: days
    };
}


// --- MAIN LOGIC ---

// 1. Get Files
const fObj = app.vault.getAbstractFileByPath(settings.objFolder);
const fRes = app.vault.getAbstractFileByPath(settings.resFolder);
const fEnt = app.vault.getAbstractFileByPath(settings.entFolder);

if (!fObj || !fRes || !fEnt) {
    new Notice("Error: Verify folder paths.");
    return;
}

const objFiles = fObj.children.filter(f => f.hasOwnProperty('extension'));
const resFiles = fRes.children.filter(f => f.hasOwnProperty('extension'));
const entFiles = fEnt.children.filter(f => f.hasOwnProperty('extension'));

// 2. Build Hierarchy Maps
// Structure: Objective -> [Results] -> [Entregas]
const mapResToObj = {}; // Key: ObjName, Value: [ResFiles]
const mapEntToRes = {}; // Key: ResName, Value: [EntFiles]

// Initialize Maps
objFiles.forEach(f => mapResToObj[f.basename] = []);
resFiles.forEach(f => mapEntToRes[f.basename] = []);

// Map Deliverables (Entregas) to Results
for (const ent of entFiles) {
    const cache = app.metadataCache.getFileCache(ent);
    if (cache?.frontmatter?.resultado) {
        const targetRes = cleanLink(cache.frontmatter.resultado);
        if (mapEntToRes[targetRes]) {
            // Attach file object to frontmatter for easier access later
            if(!cache.frontmatter.file) cache.frontmatter.file = ent; 
            mapEntToRes[targetRes].push(cache.frontmatter);
        }
    }
}

// Map Results to Objectives
for (const res of resFiles) {
    const cache = app.metadataCache.getFileCache(res);
    if (cache?.frontmatter?.objetivo) {
        const targetObj = cleanLink(cache.frontmatter.objetivo);
        if (mapResToObj[targetObj]) {
            mapResToObj[targetObj].push(res);
        }
    }
}

// 3. Draw
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

// LOOP: Objectives (Level 1)
for (let i = 0; i < objFiles.length; i++) {
    const objFile = objFiles[i];
    const myResults = mapResToObj[objFile.basename] || [];

    // --- CALCULATE HEIGHTS (Bottom-Up) ---
    const rowData = []; // Will store all pre-calculated layout data for this row
    let totalRowHeight = 0;

    // Process Results for this Objective
    for (const resFile of myResults) {
        const myEntregas = mapEntToRes[resFile.basename] || [];
        
        // A. Calculate Entregas Stack Height (Level 3)
        let entStackHeight = 0;
        const entLayouts = [];
        
        for (const entData of myEntregas) {
            const dims = getEntregaDimensions(entData);
            entLayouts.push(dims);
            entStackHeight += dims.height;
        }
        // Add gaps for entrega stack
        if (entLayouts.length > 0) entStackHeight += (entLayouts.length - 1) * settings.gapY;

        // B. Calculate Result Height (Level 2)
        // Must be at least as tall as its text, AND as tall as the Entregas stack
        const resTextData = getTextHeight(resFile.basename, maxTextWidth);
        const resFinalHeight = Math.max(settings.minHeight, resTextData.height, entStackHeight);

        rowData.push({
            resFile: resFile,
            resText: resTextData.text,
            resHeight: resFinalHeight, // The height of the Result Block
            entregas: entLayouts       // List of pre-calculated deliverable dimensions
        });

        totalRowHeight += resFinalHeight;
    }
    // Add gaps for result stack
    if (rowData.length > 0) totalRowHeight += (rowData.length - 1) * settings.gapY;

    // C. Calculate Objective Height (Level 1)
    const objTextData = getTextHeight(objFile.basename, maxTextWidth);
    const objFinalHeight = Math.max(settings.minHeight, objTextData.height, totalRowHeight);

    // --- DRAWING ---
    const color = settings.colors[i % settings.colors.length];
    ea.style.strokeColor = color;

    // 1. Draw Objective
    const objRectId = ea.addRect(settings.startX, currentY, settings.colWidth, objFinalHeight);
    const objTextId = ea.addText(settings.startX + settings.padding, currentY + settings.padding, objTextData.text);
    ea.addToGroup([objRectId, objTextId]);

    // 2. Draw Results & Entregas
    let resY = currentY;
    const resX = settings.startX + settings.colWidth + settings.gapX;
    const entX = resX + settings.colWidth + settings.gapX;

    for (const resItem of rowData) {
        // Draw Result
        const resRectId = ea.addRect(resX, resY, settings.colWidth, resItem.resHeight);
        const resTextId = ea.addText(resX + settings.padding, resY + settings.padding, resItem.resText);
        ea.addToGroup([resRectId, resTextId]);

        // Draw Entregas (Stacked next to this Result)
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
new Notice(`Generated 3-Column Matrix`);