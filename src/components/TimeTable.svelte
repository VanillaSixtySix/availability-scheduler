<script lang="ts">
    import { browser } from "$app/environment";
    import { get } from 'svelte/store';
    import { onMount, createEventDispatcher } from "svelte";

    import { DateTime } from 'luxon';
    import { previewLocal, timeZone } from '$lib/stores';

    const dispatch = createEventDispatcher();

    export let timeTable: number[] = [];
    let table: HTMLElement;

    let cells: HTMLElement[] = [];
    let isDragging: boolean = false;
    let dragStartingCell: number = -1;
    let dragStartingCellIsSelected: boolean = false;
    let lastDraggedCell: number = -1;

    let isTouchEvent: boolean = false;

    function getTargetId(event: MouseEvent | TouchEvent): number {
        let target: HTMLElement;
        if (event instanceof MouseEvent) {
            target = event.target as HTMLElement;
        } else {
            // For touch events, get the element from the point touched
            target = document.elementFromPoint(
                (event as any).touches[0].clientX,
                (event as any).touches[0].clientY
            ) as HTMLElement;
        }
        if (target == null) return -1;
        const id = parseInt(target.id);
        return id;
    }

    function startDrag(event: MouseEvent | TouchEvent) {
        if (event.type === 'mousedown') {
            isTouchEvent = true;
            setTimeout(() => {
                isTouchEvent = false;
            }, 500);
        } else if (isTouchEvent) {
            return;
        }

        const id = getTargetId(event);
        if (isNaN(id)) {
            return;
        }
        isDragging = true;
        dragStartingCell = id;

        dragStartingCellIsSelected = cells[dragStartingCell].classList.contains("selected");
        cells[id].classList.toggle("selected");
    }

    function endDrag(event: MouseEvent | TouchEvent) {
        if (!isDragging) return;

        if (event.type === 'mouseup') {
            isTouchEvent = true;
            setTimeout(() => {
                isTouchEvent = false;
            }, 500);
        } else if (isTouchEvent) {
            return;
        }

        isDragging = false;
        updateTimeTable();
    }

    function updateTimeTable() {
        timeTable = cells
            .map((cell, i) => (cell.classList.contains("selected") ? i : -1))
            .filter((id) => id !== -1);

        dispatch("message", {
            timeTable
        })
    }

    function drag(event: MouseEvent | TouchEvent) {
        if (event.type === 'mousemove') {
            isTouchEvent = true;
            setTimeout(() => {
                isTouchEvent = false;
            }, 250);
        } else if (isTouchEvent) {
            return;
        }

        if (isDragging) {
            const id = getTargetId(event);

            if (lastDraggedCell !== id) {
                const start = Math.min(dragStartingCell, id);
                const end = Math.max(dragStartingCell, id);

                const startCellRow = Math.floor(start / 7);
                const endCellRow = Math.floor(end / 7);

                const startCellCol = start % 7;
                const endCellCol = end % 7;

                const colStart = Math.min(startCellCol, endCellCol);
                const colEnd = Math.max(startCellCol, endCellCol);

                for (let i = colStart; i <= colEnd; i++) {
                    for (let j = startCellRow; j <= endCellRow; j++) {
                        const cell = cells[i + j * 7];
                        if (cell == null) return;
                        if (dragStartingCellIsSelected) {
                            cell.classList.remove("selected");
                        } else {
                            cell.classList.add("selected");
                        }
                    }
                }

                lastDraggedCell = id;
            }
        }
    }

    onMount(() => {
        cells = Array.from(table!.querySelectorAll("td"));

        // register mouse events for table dragging
        table!.addEventListener("mousedown", startDrag);
        table!.addEventListener("mouseup", endDrag);
        table!.addEventListener("mouseleave", endDrag);
        table!.addEventListener("mousemove", drag);

        // register touch events for table dragging
        table!.addEventListener("touchstart", startDrag);
        table!.addEventListener("touchend", endDrag);
        table!.addEventListener("touchmove", (event: any) => {
            if (isDragging) {
                event.preventDefault();
            }
            drag(event);
        });
    });

    $: if (browser) {
        for (let index of timeTable) {
            cells[index].classList.add("selected");
        }
    }

    previewLocal.subscribe((previewLocalValue: boolean) => {
        if (!browser) return;
        // erase all
        for (let index of timeTable) {
            cells[index].classList.remove("selected");
        }
        const nowOriginal = DateTime.local({ zone: get(timeZone) });
        const nowPreview = DateTime.local();
        // offset is hour difference
        const offset = (nowPreview.offset - nowOriginal.offset) / 60;
        const cellOffset = offset * 7;

        if (!previewLocalValue) {
            for (let i = 0; i < cells.length; i++) {
                cells[i].classList.remove("selected");
            }
        }

        for (let index of timeTable) {
            const newIndex = index + cellOffset;

            if (previewLocalValue) {
                if (newIndex < 0) {
                    cells[168 + newIndex].classList.add("selected");
                    continue;
                } else if (newIndex > 167) {
                    cells[newIndex - 168].classList.add("selected");
                    continue;
                }
                cells[index + cellOffset].classList.add("selected");
            } else {
                cells[index].classList.add("selected");
            }
        }
    });
</script>

<table bind:this={table}>
    <thead>
        <tr>
            <th />
            <th>Sun</th>
            <th>Mon</th>
            <th>Tue</th>
            <th>Wed</th>
            <th>Thu</th>
            <th>Fri</th>
            <th>Sat</th>
        </tr>
    </thead>
    <tbody>
        {#each { length: 24 } as _, i}
            <tr>
                <th>{i}</th>
                {#each { length: 7 } as _, j}
                    <td id={(7 * i + j).toString()} />
                {/each}
            </tr>
        {/each}
    </tbody>
</table>

<style>
    table {
        width: 100%;
        border-collapse: collapse;
        user-select: none;
        -webkit-user-select: none;
    }

    table td,
    table th {
        width: 50px;
        height: 20px;
        text-align: center;
    }

    table td {
        background: #eee;
        cursor: row-resize;
    }

    table td:nth-child(n + 3) {
        border-left: 1px solid #aaa;
    }

    table tr:nth-child(2n + 3) td {
        border-top: 1px solid #aaa;
    }

    /* table tr:nth-child(2n + 3) th {
        border-top: 1px solid #aaa;
    } */

    table tr:nth-child(2n + 2) td {
        border-top: 1px solid #ccc;
    }

    table td:hover {
        background: #ccc;
    }

    table :global(td.selected) {
        background: #8f8;
    }

    table :global(td.selected:hover) {
        background: #afa;
    }

    table thead th:not(:first-child) {
        position: sticky;
        top: 2rem;
        background: #fff;
        z-index: 1;
    }
</style>
