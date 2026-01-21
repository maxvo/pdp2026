/* 🎨 ROADMAP GENERATOR v5
   - Fixed: Objective Names are now LEFT ALIGNED in a strict column.
*/

{ // <--- BLOCK SCOPE START

    // 1. SETUP
    const api = ExcalidrawAutomate;
    api.reset();

    const dvPlugin = app.plugins.getPlugin("dataview");
    if (!dvPlugin || !dvPlugin.api) {
        new Notice("❌ Dataview plugin not found.");
        throw new Error("Dataview missing");
    }
    const dv = dvPlugin.api;

    // 2. SETTINGS
    const TARGET_FOLDER = "040_entrega";
    const ROW_HEIGHT = 120;
    const DAY_WIDTH = 20;
    
    // COLUMN SETTINGS
    const LABEL_WIDTH = 250;  // How wide the text box is
    const LABEL_X_START = -280; // Where the text starts (Left edge)

    // 3. GET DATA
    const query = `"${TARGET_FOLDER}"`;
    const pages = dv.pages(query).where(p => p.startDate && p.dueDate);

    if (pages.length === 0) {
        new Notice(`⚠️ No projects found in ${TARGET_FOLDER}`);
    } else {

        // 4. PROCESS DATA
        let minDate = new Date("2099-12-31");
        const groups = {};
        const colors = ["#ffc9c9", "#b2f2bb", "#a5d8ff", "#ffec99", "#e5dbff"];

        const cleanName = (val) => {
            if (!val) return "No Objective";
            let str = val.path || (Array.isArray(val) ? val[0]?.path : String(val));
            if (!str) return "No Objective";
            return str.split("/").pop().replace(".md", "").trim();
        };

        for (let p of pages) {
            const tDate = new Date(p.startDate.ts || p.startDate);
            if (tDate < minDate) minDate = tDate;

            const objName = cleanName(p.resultado);
            if (!groups[objName]) groups[objName] = [];
            groups[objName].push(p);
        }

        // 5. DRAWING LOOP
        let row = 0;
        
        for (const groupName in groups) {
            const y = row * ROW_HEIGHT;
            const color = colors[row % colors.length];

            // A. DRAW OBJECTIVE LABEL (Left Aligned)
            api.style.strokeColor = "#000000";
            api.style.strokeWidth = 1;
            api.style.fillStyle = "solid";
            
            api.addText(LABEL_X_START, y + 40, groupName, { 
                width: LABEL_WIDTH, 
                textAlign: "left",  // <--- CHANGED TO LEFT
                fontSize: 16
            });

            // B. DRAW DIVIDER LINE
            // Line separates the text column from the chart
            const lineX = LABEL_X_START + LABEL_WIDTH + 10;
            const lineStart = [ lineX, y + 100 ];
            const lineEnd = [ 1000, y + 100 ]; // Extend line far to the right
            api.addLine([ lineStart, lineEnd ]);

            // C. DRAW TASKS
            for (let p of groups[groupName]) {
                const start = new Date(p.startDate.ts || p.startDate);
                const end = new Date(p.dueDate.ts || p.dueDate);

                const startDiff = (start - minDate) / (1000 * 3600 * 24);
                const duration = (end - start) / (1000 * 3600 * 24);

                // Offset X by the end of the Label Column so chart starts after text
                const chartStartX = lineX + 20; 
                const x = chartStartX + (startDiff * DAY_WIDTH);
                const w = Math.max(duration * DAY_WIDTH, 60);

                // Box
                api.style.backgroundColor = color;
                api.style.fillStyle = "hachure";
                api.style.strokeWidth = 2;
                api.addRect(x, y + 20, w, 60);

                // Text
                api.style.textAlign = "center";
                api.style.fontSize = 14;
                let label = p.file.name.replace(".md", "");
                if (label.length > 25) label = label.substring(0, 23) + "..";
                
                api.addText(x + (w/2), y + 40, label, { width: w, textAlign: "center" });
            }
            row++;
        }

        // 6. FINALIZE
        await api.create({
            view: "active",
            shouldRestoreMainObject: true
        });

        new Notice("✅ Roadmap Generated!");
    }

} // <--- BLOCK SCOPE END