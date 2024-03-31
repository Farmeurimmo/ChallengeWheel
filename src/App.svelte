<script>
    import {afterUpdate} from "svelte";
    import * as d3 from "d3";
    import wheel from './wheel.json';

    const width = 500;
    const height = 500;

    const colorMapping = {
        "Malus": "purple",
        "Bonus": "green",
        "Malus extrême": "red",
        "Neutre": "yellow"
    };

    let svg;
    let data = [];

    // Parse the JSON data into the required format
    Object.keys(wheel).forEach(key => {
        wheel[key].forEach(item => {
            data.push({
                label: item.description,
                value: parseInt(item.probability),
                category: key // Add the category here
            });
        });
    });

    // Randomize the order of the items
    data = d3.shuffle(data);

    function getItem() {
        let items = [];
        //for data, calculate the angle range for each item and push it to the items array, those angles should be in °
        data.forEach((d, i) => {
            const startAngle = i * 360 / data.length;
            const endAngle = (i + 1) * 360 / data.length;
            items.push({
                startAngle,
                endAngle,
                label: d.label,
                value: d.value,
                category: d.category
            });
        });
        return items;
    }

    let wheelElement;
    let resultText = "...";
    let wheelCreated = false;
    let spinning = false;
    let items = [];

    afterUpdate(() => {
        if (!wheelCreated) {
            createWheel();
            wheelCreated = true;
        }
    });

    function createWheel() {
        // Create the pie chart
        const radius = Math.min(width, height) / 2;

        d3.select(wheelElement).select("svg").remove();

        svg = d3.select(wheelElement)
            .append("svg")
            .attr("width", width)
            .attr("height", height)
            .append("g")
            .attr("transform", "translate(" + width / 2 + "," + height / 2 + ")");

        const pie = d3.pie().value(d => d.value);
        const arc = d3.arc().innerRadius(0).outerRadius(radius);

        items = pie(getItem());
        items.forEach((d, i) => {
            d.category = data[i].category;
            d.label = data[i].label;
        });

        svg.append("defs")
            .append("pattern")
            .attr("id", "malusPattern")
            .attr("width", "100")
            .attr("height", "100")
            .attr("patternUnits", "userSpaceOnUse")
            .append("image")
            .attr("xlink:href", "/malus.jpg")
            .attr("width", "400")
            .attr("height", "400")
            .attr("preserveAspectRatio", "none");

        const path = svg.selectAll("path")
            .data(pie(items))
            .enter()
            .append("path")
            .attr("d", arc)
            .attr("fill", d => d.data.category === "Malus" ? "url(#malusPattern)" : colorMapping[d.data.category])
            .attr("stroke", "white")
            .attr("stroke-width", "2px")
            .on("mouseover", function (event, d) {
                d3.select("#tooltip")
                    .style("left", (event.pageX + 10) + "px")
                    .style("top", (event.pageY - 10) + "px")
                    .style("visibility", "visible")
                    .text(d.data.label);
            })
            .on("mouseout", function () {
                d3.select("#tooltip")
                    .style("visibility", "hidden");
            });

        path.append("title")
            .text(d => d.data.label);
    }

    function spinWheel() {
        if (spinning) {
            return;
        }
        spinning = true;
        resultText = "...";
        // take a random item from the items list taking into account the probability
        let randomItem;
        let total = d3.sum(items, d => d.value);
        let random = Math.random() * total;
        let sum = 0;
        items.forEach(d => {
            sum += d.value;
            if (sum >= random && !randomItem) {
                randomItem = d;
            }
        });
        if (!randomItem) {
            console.error("No item selected");
            return;
        }
        // @ts-ignore
        let targetAngle = (randomItem.startAngle + randomItem.endAngle) / 2;
        let rotation = 5 * 360 - targetAngle * 180 / Math.PI;


        svg.transition()
            .duration(4000)
            .attrTween("transform", () => {
                const interpolate = d3.interpolateString("rotate(0)", `rotate(${rotation})`);
                return t => {
                    return `translate(${width / 2}, ${height / 2}) ${interpolate(t)}`;
                };
            })
            .on("end", () => {
                spinning = false;
                let items_copy = items.map(d => ({...d}));
                const piPosition = rotation % 360 + targetAngle % 360;
                items_copy.forEach((d) => {
                    d.startAngle += rotation % 360;
                    d.endAngle += rotation % 360;
                });
                const winningItem = items_copy.find(d => d.startAngle <= piPosition && d.endAngle >= piPosition);
                resultText = winningItem.label;
            });
    }
</script>

<main class="flex flex-col w-full h-full overflow-hidden">
    <section id="header">
        <h1>Roue des défis</h1>
    </section>
    <section class="mt-20 flex justify-center items-center h-[500px] flex-col" id="wheel">
        <p class="text-3xl mt-5 font-bold">Résultat</p>
        <p class="text-xl" id="result">{resultText}</p>
        <p id="arrow" class="mb-2 text-6xl font-extrabold">&darr;</p>
        <div bind:this={wheelElement}></div>
        <div class="hidden font-bold text-xl" id="tooltip"></div>
    </section>
    <button class="mt-28" class:spinning on:click={spinWheel}>Lancer la roue</button>
    <section class="mt-40" id="details">
        <p>Site réalisé par <a href="https://farmeurimmo.fr">Farmeurimmo</a> | Open Source sur <a href="https://github.com/Farmeurimmo/ChallengeWheel">GitHub</a></p>
    </section>
</main>