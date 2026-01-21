/* * Excalidraw Script: Objective & Key Results (Fixed)
* Description: Handles YAML links correctly (List or String).
*/

// --- CONFIGURATION ---
const settings = {
    objFolder: "020_objetivo",
    resFolder: "030_resultado",
    
    colWidth: 300,        
    colGap: 50,           
    
    minHeight: 60,        
    padding: 10,          
    itemGap: 10,          
    
    startX: 0,
    startY: 0,
    
    fontSize: 20,
    fontFamily: 2,
    
    colors: [
        "#4a6fa5", // Muted Slate Blue
        "#5c8a8a"  // Muted Sage Teal
    ]
};

// --- HELPER: Word Wrap ---
function wrapText(text, maxWidth) {
    if (!text) return ""; // Safety check
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

// --- HELPER: Get Height ---
function getTextHeight(text, maxWidth) {
    const wrapped = wrapText(text, maxWidth);
    const metrics = ea.measureText(wrapped);
    return {
        text: wrapped,
        height: metrics.height + (settings.padding * 2)
    };
}

// --- HELPER: Clean Link (FIXED) ---
function cleanLink(linkValue) {
    if (!linkValue) return "";
    
    // 1. If it's an array (List), take the first item
    let raw = Array.isArray(linkValue) ? linkValue[0] : linkValue;
    
    // 2. Ensure it's a string before processing
    if (typeof raw !== 'string') {
        // Fallback: convert to string or return empty if invalid
        return String(raw) || "";
    }

    // 3. Clean wiki brackets
    return raw.replace(/\[\[|\]\]/g, "").split("|")[0];
}

// --- MAIN LOGIC ---

// 1. Load Files
const folderObj = app.vault.getAbstractFileByPath(settings.objFolder);
const folderRes = app.vault.getAbstractFileByPath(settings.resFolder);

if (!folderObj || !folderRes) {
    new Notice("Error: One of the folders was not found.");
    return;
}

const objFiles = folderObj.children.filter(f => f.hasOwnProperty('extension'));
const resFiles = folderRes.children.filter(f => f.hasOwnProperty('extension'));

// 2. Map Results to Objectives
const relationships = {};

objFiles.forEach(f => relationships[f.basename] = []);

for (const rFile of resFiles) {
    const cache = app.metadataCache.getFileCache(rFile);
    if (cache && cache.frontmatter && cache.frontmatter.objetivo) {
        
        // Use the FIXED function here
        const targetObjName = cleanLink(cache.frontmatter.objetivo);
        
        if (relationships[targetObjName]) {
            relationships[targetObjName].push(rFile);
        }
    }
}

// 3. Draw Elements
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

// Loop through Objectives
for (let i = 0; i < objFiles.length; i++) {
    const objFile = objFiles[i];
    const relatedResFiles = relationships[objFile.basename] || [];
    
    // --- STEP A: CALCULATE HEIGHTS ---
    const objTextData = getTextHeight(objFile.basename, maxTextWidth);
    let objOnlyHeight = Math.max(settings.minHeight, objTextData.height);
    
    let resStackHeight = 0;
    const resDataList = [];
    
    for (const rFile of relatedResFiles) {
        const rTextData = getTextHeight(rFile.basename, maxTextWidth);
        const rHeight = Math.max(settings.minHeight, rTextData.height);
        
        resDataList.push({
            file: rFile,
            text: rTextData.text,
            height: rHeight
        });
        
        resStackHeight += rHeight;
    }
    
    if (relatedResFiles.length > 0) {
        resStackHeight += (relatedResFiles.length - 1) * settings.itemGap;
    }

    const finalRowHeight = Math.max(objOnlyHeight, resStackHeight);

    // --- STEP B: DRAWING ---
    const color = settings.colors[i % settings.colors.length];
    ea.style.strokeColor = color;
    
    // Draw Objective (Column 1)
    // We stretch this to finalRowHeight
    const objRectId = ea.addRect(settings.startX, currentY, settings.colWidth, finalRowHeight);
    const objTextId = ea.addText(settings.startX + settings.padding, currentY + settings.padding, objTextData.text);
    ea.addToGroup([objRectId, objTextId]);
    
    // Draw Results (Column 2)
    let resCurrentY = currentY;
    const resX = settings.startX + settings.colWidth + settings.colGap;
    
    for (const resData of resDataList) {
        const resRectId = ea.addRect(resX, resCurrentY, settings.colWidth, resData.height);
        const resTextId = ea.addText(resX + settings.padding, resCurrentY + settings.padding, resData.text);
        ea.addToGroup([resRectId, resTextId]);
        
        resCurrentY += resData.height + settings.itemGap;
    }

    currentY += finalRowHeight + settings.itemGap;
}

await ea.addElementsToView(true, true, true);
new Notice(`Generated Matrix: ${objFiles.length} Objectives.`);