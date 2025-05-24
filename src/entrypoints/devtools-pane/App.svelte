<script lang="ts">
    import { onMount } from "svelte";
    
    let element: string = $state("");
    let elementDetails: ElementDetails | null = $state(null);

    type ElementDetails = {
        tagName: string;
        id: string;
        className: string;
        textContent: string;
        attributes: { name: string; value: string }[];
        computedStyles: Record<string, string>;
        layoutMode?: {
            mode: string;
            variant: string;
            details: string;
        };
    };

    function evaluateCssLayoutMode(element: ElementDetails) {
        // Check for positioned layout
        if (
            element.computedStyles.position === "absolute" ||
            element.computedStyles.position === "fixed" ||
            element.computedStyles.position === "sticky"
        ) {
            return {
                mode: "Positioned",
                variant: element.computedStyles.position,
                details:
                    "Out-of-flow positioning scheme where the box is removed from normal flow and positioned relative to its containing block. Uses properties like top, left, right, bottom, and z-index. Creates a new positioning context for descendants.",
            };
        }

        // Check for parent-defined layout modes
        if (
            element.computedStyles.display === "flex" ||
            element.computedStyles.display === "inline-flex"
        ) {
            return {
                mode: "Flexbox",
                variant: element.computedStyles.display,
                details:
                    "A flexible box layout that establishes an independent flex formatting context. Child elements become flex items arranged along main and cross axes. Uses properties like flex-direction, justify-content, align-items, and flex properties on children.",
            };
        }

        if (
            element.computedStyles.display === "grid" ||
            element.computedStyles.display === "inline-grid"
        ) {
            return {
                mode: "Grid",
                variant: element.computedStyles.display,
                details:
                    "A grid-based layout that establishes an independent grid formatting context. Child elements become grid items positioned within defined grid cells. Uses properties like grid-template-columns, grid-template-rows, grid-area, and grid-placement properties.",
            };
        }

        if (
            element.computedStyles.display === "table" ||
            element.computedStyles.display === "inline-table" ||
            element.computedStyles.display === "table-row" ||
            element.computedStyles.display === "table-cell"
        ) {
            return {
                mode: "Table",
                variant: element.computedStyles.display,
                details: "Table-based layout that establishes a table formatting context. Elements behave like table parts (rows, cells, etc.). Can create anonymous boxes to ensure proper table structure. Useful for tabular data but less flexible than modern layout methods.",
            };
        }

        // Check for float layout
        if (
            element.computedStyles.float &&
            element.computedStyles.float !== "none"
        ) {
            return {
                mode: "Float",
                variant: element.computedStyles.float,
                details:
                    "A partially out-of-flow positioning scheme where the box is first laid out in normal flow, then taken out and shifted left or right. Content flows around floated elements. Can be cleared with the clear property. Creates a new block formatting context when contained with display: flow-root.",
            };
        }

        // Default to Flow layout with block or inline variants
        if (element.computedStyles.display === "block") {
            return {
                mode: "Flow",
                variant: "block",
                details:
                    "Standard block-level flow layout in the normal flow - elements stack vertically in the block direction. Creates a block formatting context containing block-level boxes. Takes up the full width available and creates line breaks before and after.",
            };
        }

        if (element.computedStyles.display === "inline") {
            return {
                mode: "Flow",
                variant: "inline",
                details:
                    "Standard inline-level flow layout in the normal flow - elements flow horizontally with text in the inline direction. Participates in an inline formatting context. Does not create line breaks before or after. Margins/paddings only apply horizontally, not vertically.",
            };
        }

        if (element.computedStyles.display === "inline-block") {
            return {
                mode: "Flow",
                variant: "inline-block",
                details:
                    "Hybrid flow layout - behaves like a block container for its contents but flows inline like an inline element. Creates a block formatting context while participating in the surrounding inline formatting context. Supports vertical margins/padding unlike inline elements.",
            };
        }

        // Fallback for other display values
        return {
            mode: "Flow",
            variant: element.computedStyles.display || "unknown",
            details: "Default document flow layout. Part of the normal flow positioning scheme where boxes are laid out one after another in the document's writing mode. May participate in block or inline formatting contexts depending on the display value.",
        };
    }

    function update() {
        browser.devtools.inspectedWindow.eval(
            `(function() {
                const el = $0;
                if (!el) return { error: 'No element selected' };

                return {
                    tagName: el.tagName,
                    id: el.id,
                    className: el.className,
                    textContent: el.textContent.substring(0, 100),
                    attributes: Array.from(el.attributes).map(attr => ({
                        name: attr.name,
                        value: attr.value
                    })),
                    computedStyles: (() => {
                        const styles = window.getComputedStyle(el);
                        const result = {};
                        for (let i = 0; i < styles.length; i++) {
                            const name = styles[i];
                            result[name] = styles.getPropertyValue(name);
                        }
                        return result;
                    })(),


                };
            })()`,
            (result: ElementDetails, isException) => {
                if (isException) {
                    console.error("Error:", result);
                    element = "Error getting element";
                } else {
                    element = result.tagName
                        ? result.tagName.toLowerCase()
                        : "No element selected";
                    elementDetails = result;

                    // Evaluate layout mode if we have element details
                    if (elementDetails) {
                        elementDetails.layoutMode =
                            evaluateCssLayoutMode(elementDetails);
                    }
                }
            },
        );

        // // inject a content script function
        // browser.devtools.inspectedWindow.eval(
        //     "console.log('Selected Element in console:', $0)",
        //     {
        //         useContentScriptContext: false,
        //     },
        // );
    }

    let layoutMode = $derived(elementDetails ? evaluateCssLayoutMode(elementDetails) : null);
    
    onMount(() => {
        update();
        
        // Set up listener for element selection changes
        browser.devtools.panels.elements.onSelectionChanged.addListener(() => {
            update();
        });
    });
</script>

<main>
    {#if element}
        <div class="element-info">
            <h2>{element}</h2>
            {#if elementDetails}
                {#if layoutMode}
                <div class="layout-summary">
                    <p class="layout-mode-type">
                        <strong>Layout Algorithm:</strong>
                        {layoutMode.mode}
                    </p>
                    <p class="layout-variant">
                        <strong>Variant:</strong>
                        {layoutMode.variant}
                    </p>
                    <p class="layout-description">
                        {layoutMode.details}
                    </p>
                </div>
                {/if}
                <details>
                    <summary>Details</summary>
                    <div class="details">
                        <h3>Element Details</h3>
                        {#if elementDetails.id}
                            <p><strong>ID:</strong> {elementDetails.id}</p>
                        {/if}
                        {#if elementDetails.className}
                            <p>
                                <strong>Classes:</strong>
                                {elementDetails.className}
                            </p>
                        {/if}
                        {#if elementDetails.attributes && elementDetails.attributes.length > 0}
                            <div class="attributes">
                                <h4>Attributes</h4>
                                <ul>
                                    {#each elementDetails.attributes as attr}
                                        <li>
                                            <strong>{attr.name}:</strong>
                                            {attr.value}
                                        </li>
                                    {/each}
                                </ul>
                            </div>
                        {/if}
                        {#if elementDetails.textContent}
                            <p>
                                <strong>Text:</strong>
                                {elementDetails.textContent}...
                            </p>
                        {/if}

                        {#if elementDetails.layoutMode}
                            <div class="layout-mode">
                                <h4>CSS Layout Mode</h4>
                                <p>
                                    <strong>Mode:</strong>
                                    {elementDetails.layoutMode.mode}
                                </p>
                                <p>
                                    <strong>Variant:</strong>
                                    {elementDetails.layoutMode.variant}
                                </p>
                                <p>
                                    <strong>Details:</strong>
                                    {elementDetails.layoutMode.details}
                                </p>
                            </div>
                        {/if}
                    </div>
                    <pre>{JSON.stringify(elementDetails, null, 2)}</pre>
                </details>
            {/if}
        </div>
    {/if}
</main>

<style>
    .layout-mode {
        margin-top: 1rem;
        padding: 0.5rem;
        border: 1px solid #ddd;
        border-radius: 4px;
        background-color: #f8f8f8;
    }

    .layout-mode h4 {
        margin-top: 0;
        color: #333;
    }

    pre {
        max-height: 300px;
        overflow: auto;
        background-color: #f5f5f5;
        padding: 0.5rem;
        border-radius: 4px;
    }
</style>
