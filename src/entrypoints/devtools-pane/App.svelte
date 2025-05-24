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
                    "Out-of-flow positioning scheme where the box is removed from normal flow and positioned relative to its containing block. Uses properties like top, left, right, bottom, and z-index. Creates a new positioning context for descendants. <a href='https://developer.mozilla.org/en-US/docs/Web/CSS/CSS_positioned_layout' target='_blank'>MDN: Positioned Layout</a>",
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
                    "A flexible box layout that establishes an independent flex formatting context. Child elements become flex items arranged along main and cross axes. Uses properties like flex-direction, justify-content, align-items, and flex properties on children. <a href='https://developer.mozilla.org/en-US/docs/Web/CSS/CSS_flexible_box_layout' target='_blank'>MDN: Flexbox</a>",
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
                    "A grid-based layout that establishes an independent grid formatting context. Child elements become grid items positioned within defined grid cells. Uses properties like grid-template-columns, grid-template-rows, grid-area, and grid-placement properties. <a href='https://developer.mozilla.org/en-US/docs/Web/CSS/CSS_grid_layout' target='_blank'>MDN: Grid Layout</a>",
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
                details:
                    "Table-based layout that establishes a table formatting context. Elements behave like table parts (rows, cells, etc.). Can create anonymous boxes to ensure proper table structure. Useful for tabular data but less flexible than modern layout methods. <a href='https://developer.mozilla.org/en-US/docs/Web/CSS/CSS_table' target='_blank'>MDN: Table Layout</a>",
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
                    "A partially out-of-flow positioning scheme where the box is first laid out in normal flow, then taken out and shifted left or right. Content flows around floated elements. Can be cleared with the clear property. Creates a new block formatting context when contained with display: flow-root. <a href='https://developer.mozilla.org/en-US/docs/Web/CSS/float' target='_blank'>MDN: Floats</a>",
            };
        }

        // Default to Flow layout with block or inline variants
        if (element.computedStyles.display === "block") {
            return {
                mode: "Flow",
                variant: "block",
                details:
                    "Standard block-level flow layout in the normal flow - elements stack vertically in the block direction. Creates a block formatting context containing block-level boxes. Takes up the full width available and creates line breaks before and after. <a href='https://developer.mozilla.org/en-US/docs/Web/CSS/CSS_display/Block_formatting_context' target='_blank'>MDN: Block Formatting</a>",
            };
        }

        if (element.computedStyles.display === "inline") {
            return {
                mode: "Flow",
                variant: "inline",
                details:
                    "Standard inline-level flow layout in the normal flow - elements flow horizontally with text in the inline direction. Participates in an inline formatting context. Does not create line breaks before or after. Margins/paddings only apply horizontally, not vertically. <a href='https://developer.mozilla.org/en-US/docs/Web/CSS/CSS_display#inline-level_elements' target='_blank'>MDN: Inline Formatting</a>",
            };
        }

        if (element.computedStyles.display === "inline-block") {
            return {
                mode: "Flow",
                variant: "inline-block",
                details:
                    "Hybrid flow layout - behaves like a block container for its contents but flows inline like an inline element. Creates a block formatting context while participating in the surrounding inline formatting context. Supports vertical margins/padding unlike inline elements. <a href='https://developer.mozilla.org/en-US/docs/Web/CSS/display-inline-block' target='_blank'>MDN: Inline-Block</a>",
            };
        }

        // Fallback for other display values
        return {
            mode: "Flow",
            variant: element.computedStyles.display || "unknown",
            details:
                "Default document flow layout. Part of the normal flow positioning scheme where boxes are laid out one after another in the document's writing mode. May participate in block or inline formatting contexts depending on the display value. <a href='https://developer.mozilla.org/en-US/docs/Web/CSS/CSS_display/Visual_formatting_model' target='_blank'>MDN: Visual Formatting Model</a>",
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

    let layoutMode = $derived(
        elementDetails ? evaluateCssLayoutMode(elementDetails) : null,
    );

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
                            {@html layoutMode.details}
                        </p>
                    </div>
                {/if}
            {/if}
        </div>
    {/if}
</main>

<style>
    :global(.layout-description a) {
        color: #0366d6;
        text-decoration: none;
    }

    :global(.layout-description a:hover) {
        text-decoration: underline;
    }
</style>
