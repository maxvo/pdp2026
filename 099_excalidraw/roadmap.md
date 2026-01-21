```dataviewjs
// 1. Get all files from your "Projects" folder
const projects = dv.pages('"040_entrega"'); 

// 2. Build the Mermaid string
let mermaidCode = "gantt\n";
mermaidCode += "    title Project Roadmap 2026\n";
mermaidCode += "    dateFormat YYYY-MM-DD\n";
mermaidCode += "    section Active Projects\n";

for (let p of projects) {
    // Only add if dates exist in properties
    if (p.startDate && p.dueDate) {
        mermaidCode += `    ${p.file.name} : ${p.startDate}, ${p.dueDate}\n`;
    }
}

// 3. Render it as an editable Excalidraw element
// You can use the 'Excalidraw: Insert Mermaid' command and paste the output of this script.
dv.paragraph("### Generated Mermaid Code (Copy this):");
dv.paragraph("```\n" + mermaidCode + "\n```");
```
```dataviewjs
// 1. Settings
const folderPath = '"040_entrega"';
const projects = dv.pages(folderPath)
    .where(p => p.startDate && p.dueDate)
    .sort(p => p.startDate, 'asc');

// 2. Grouping Logic
const groups = {};

for (let p of projects) {
    let finalGroupName = "No Objective";
    
    // Step A: Find the 'Result' Note
    if (p.resultado) {
        let path = "";
        // Handle if 'resultado' is a Link, a Path String, or an Array
        if (p.resultado.path) path = p.resultado.path;
        else if (typeof p.resultado === 'string') path = p.resultado;
        else if (Array.isArray(p.resultado) && p.resultado[0]) path = p.resultado[0].path || p.resultado[0];

        if (path) {
            const resPage = dv.page(path);
            
            // Step B: Get the 'Objective' and CLEAN IT immediately
            if (resPage && resPage.objetivo) {
                const rawObj = resPage.objetivo;
                let rawString = "";

                // Force extraction of the file path/name
                if (rawObj.path) rawString = rawObj.path; // It is a link
                else if (Array.isArray(rawObj)) rawString = rawObj[0].path || String(rawObj[0]); 
                else rawString = String(rawObj); // It is a string

                // Step C: Normalize (Remove folders, extensions, and whitespace)
                // "Folder/My Objective.md" -> "My Objective"
                finalGroupName = rawString.split('/').pop().replace('.md', '').trim();
            }
        }
    }

    // Add to group (Key is now guaranteed to be a clean string)
    if (!groups[finalGroupName]) groups[finalGroupName] = [];
    groups[finalGroupName].push(p);
}

// 3. Build Mermaid String
let mm = "gantt\n";
mm += "    title Roadmap 040_entrega\n";
mm += "    dateFormat YYYY-MM-DD\n";
mm += "    axisFormat %m-%d\n";
mm += "    todayMarker off\n\n"; 

for (const groupName in groups) {
    mm += `    section ${groupName}\n`;
    
    for (let p of groups[groupName]) {
        // Clean the Project Name for the chart label
        const cleanName = p.file.name.replace(/[\[\]\:\#]/g, ''); 
        mm += `    ${cleanName} : ${p.startDate}, ${p.dueDate}\n`;
    }
}

dv.paragraph("### Generated Mermaid Code");
dv.paragraph(mm + "\n```");
```

```dataviewjs
// 1. Settings
const folderPath = '"040_entrega"'; 
const projects = dv.pages(folderPath)
    .where(p => p.startDate && p.dueDate)
    .sort(p => p.startDate, 'asc');

// 2. Grouping Logic
const groups = {};
for (let p of projects) {
    let groupName = "Uncategorized";
    if (p.resultado) {
        let raw = "";
        if (p.resultado.path) raw = p.resultado.path;
        else if (Array.isArray(p.resultado)) raw = p.resultado[0].path || String(p.resultado[0]);
        else raw = String(p.resultado);
        groupName = raw.split('/').pop().replace('.md', '').trim();
    }
    if (!groups[groupName]) groups[groupName] = [];
    groups[groupName].push(p);
}

// 3. Build Mermaid Flowchart
let mermaid = "graph LR\n";

// Style: Dark box with light text
mermaid += "    classDef project fill:#202020,stroke:#888,color:#fff,stroke-width:2px,text-align:center\n";

let nodeId = 0;

for (const groupName in groups) {
    // Clean Group ID
    const groupId = "g" + nodeId + "X"; 
    
    mermaid += `    subgraph ${groupId} ["${groupName}"]\n`;
    mermaid += `    direction LR\n`; 
    
    let previousNodeId = null;
    
    for (let p of groups[groupName]) {
        const currentId = `n${nodeId}`;
        const cleanName = p.file.name.replace(/["']/g, ''); 
        
        // --- DATE FORMATTING ---
        // We access the Luxon date object directly
        // Format: dd/MM (e.g., 05/01)
        const start = p.startDate.toFormat ? p.startDate.toFormat('dd/MM') : p.startDate;
        const end = p.dueDate.toFormat ? p.dueDate.toFormat('dd/MM') : p.dueDate;
        
        // HTML Line Break (<br/>) puts dates on a new line inside the box
        const label = `${cleanName}<br/>📅 ${start} - ${end}`;
        
        // Add Node
        mermaid += `        ${currentId}["${label}"]:::project\n`;
        
        // Sequence Link
        if (previousNodeId) {
            mermaid += `        ${previousNodeId} --> ${currentId}\n`;
        }
        
        previousNodeId = currentId;
        nodeId++;
    }
    mermaid += "    end\n"; 
}

dv.paragraph("### Copy for Excalidraw:");
dv.paragraph("````text\n```mermaid\n" + mermaid + "\n```\n````");
```
