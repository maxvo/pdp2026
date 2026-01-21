(async () => {
    // 1. INITIALIZATION & SAFETY CHECKS
    const ea = ExcalidrawAutomate;
    ea.reset();

    // Check for Dataview API safely
    const dataviewPlugin = app.plugins.getPlugin("dataview");
    if (!dataviewPlugin || !dataviewPlugin.api) {
        new Notice("❌ Dataview plugin not found! Please install it.");
        return;
    }
    const dv = dataviewPlugin.api;

    // 2. SETTINGS
    // Your folder name (no extra quotes needed here, we handle it below)
    const TARGET_FOLDER = "040_entrega"; 
    
    // 3. FETCH DATA
    // We manually construct the query string: '"040_entrega"'
    const query = `"${TARGET_FOLDER}"`;
    const pages = dv.pages(query).where(p => p.startDate && p.dueDate);

    if (pages.length === 0) {
        new Notice(`⚠️ No projects found in folder: ${TARGET_FOLDER}`);
        return;
    }

    // 4. PREPARE DATA GROUPS
    let minDate = new Date("2099-12-31");
    const groups = {};
    const colors = ["#ffc9c9", "#b2f2bb", "#a5d8ff", "#ffec99", "#e5dbff"];

    // Helper: Clean objective name
    const cleanObjective = (page) => {
        if (!page.resultado) return "No Objective";
        
        let pathStr = "";
        // Handle direct links, paths, or arrays of links
        if (page.resultado.path) pathStr = page.resultado.path;
        else if (Array.isArray(page.resultado)) pathStr = page.resultado[0]?.path || "";
        else pathStr = String(page.resultado);

        if (!pathStr) return "No Objective";

        // Extract filename from path (e.g. "Folder/Objective.md" -> "Objective")
        const parts = pathStr.split("/");
        return parts[parts.length - 1].replace(".md", "").trim();
    };

    for (let p of pages) {
        // Find global start date for the timeline
        const tDate = new Date(p.startDate.ts || p.startDate);
        if (tDate < minDate) minDate = tDate;

        const objName = cleanObjective(p);
        if (!groups[objName]) groups[objName] = [];
        groups[objName].push(p);
    }

    // 5. DRAWING LOGIC
    let currentRow = 0;
    const ROW_HEIGHT = 120;
    const DAY_WIDTH = 20; // 20px per day

    for (const groupName in groups) {
        const yBase = currentRow * ROW_HEIGHT;
        const color = colors[currentRow % colors.length];

        // A. Draw Swimlane Label (Left side)
        ea.style.strokeColor = "#000000";
        ea.style.strokeWidth = 1;
        ea.style.fillStyle = "solid";
        ea.style.textAlign = "right";
        ea.addText(-200, yBase + 40, groupName, {fontSize: 20});

        // B. Draw Divider Line
        ea.drawLine([[-220, yBase + 100], [1200, yBase + 100]]);

        // C. Draw Task Bars
        for (let p of groups[groupName]) {
            const start = new Date(p.startDate.ts || p.startDate);
            const end = new Date(p.dueDate.ts || p.dueDate);
            
            // Calculate X Position and Width
            const startDiff = (start - minDate) / (1000 * 3600 * 24); // Days from start
            const duration = (end - start) / (1000 * 3600 * 24);      // Duration in days
            
            const x = startDiff * DAY_WIDTH;
            const w = Math.max(duration * DAY_WIDTH, 60); // Minimum width 60px

            // Box Style
            ea.style.backgroundColor = color;
            ea.style.fillStyle = "hachure";
            ea.style.strokeWidth = 2;
            
            // Add Box
            const boxId = ea.addRect(x, yBase + 20, w, 60);
            
            // Add Text Label
            ea.style.textAlign = "center";
            ea.style.fontSize = 14;
            // Shorten name if it's too long
            let label = p.file.name;
            if (label.length > 20) label = label.substring(0, 18) + "..";
            
            ea.addText(x + (w/2), yBase + 40, label, {width: w});
            
            // Optional: Link the box to the note
            // ea.elementsDict[boxId].link = `[[${p.file.path}]]`;
        }
        currentRow++;
    }

    // 6. FINALIZE
    await ea.create({
        view: "active",
        shouldRestoreMainObject: true
    });

    new Notice("✅ Roadmap generated successfully!");
})();
