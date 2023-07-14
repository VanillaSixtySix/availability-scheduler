<script lang="ts">
    import { Button, Input } from "sveltestrap";
    import TimeTable from "../components/TimeTable.svelte";
    import { onMount } from "svelte";
    import { browser } from "$app/environment";
    import { timeZone, previewLocal } from '$lib/stores';
    import { get } from 'svelte/store';

    interface TimeZone {
        value: string;
        viewValue: string;
    }

    const timezones: TimeZone[] = [
        {
            value: "Pacific/Honolulu",
            viewValue: "UTC-10 - Hawaiian Standard Time (HST)",
        },
        {
            value: "America/Anchorage",
            viewValue: "UTC-9 - Alaska Standard Time (AKST)",
        },
        {
            value: "America/Los_Angeles",
            viewValue: "UTC-8 - Pacific Standard Time (PST)",
        },
        {
            value: "America/Denver",
            viewValue: "UTC-7 - Mountain Standard Time (MST)",
        },
        {
            value: "America/Chicago",
            viewValue: "UTC-6 - Central Standard Time (CST)",
        },
        {
            value: "America/New_York",
            viewValue: "UTC-5 - Eastern Standard Time (EST)",
        },
        {
            value: "America/Puerto_Rico",
            viewValue: "UTC-4 - Atlantic Standard Time (AST)",
        },
        {
            value: "America/Sao_Paulo",
            viewValue: "UTC-3 - Brasília Standard Time (BRT)",
        },
        {
            value: "Atlantic/Cape_Verde",
            viewValue: "UTC-1 - Cape Verde Time (CVT)",
        },
        {
            value: "Europe/London",
            viewValue: "UTC - Greenwich Mean Time (GMT)",
        },
        {
            value: "Europe/Paris",
            viewValue: "UTC+1 - Central European Time (CET)",
        },
        {
            value: "Europe/Moscow",
            viewValue: "UTC+3 - Moscow Standard Time (MSK)",
        },
        {
            value: "Asia/Dubai",
            viewValue: "UTC+4 - Gulf Standard Time (GST)"
        },
        {
            value: "Asia/Karachi",
            viewValue: "UTC+5 - Pakistan Standard Time (PKT)",
        },
        {
            value: "Asia/Dhaka",
            viewValue: "UTC+6 - Bangladesh Standard Time (BDT)",
        },
        {
            value: "Asia/Bangkok",
            viewValue: "UTC+7 - Indochina Time (ICT)"
        },
        {
            value: "Asia/Shanghai",
            viewValue: "UTC+8 - China Standard Time (CST)",
        },
        {
            value: "Australia/Perth",
            viewValue: "UTC+8 - Australian Western Standard Time (AWST)",
        },
        {
            value: "Asia/Tokyo",
            viewValue: "UTC+9 - Japan Standard Time (JST)"
        },
        {
            value: "Australia/Sydney",
            viewValue: "UTC+10 - Australian Eastern Standard Time (AEST)",
        },
        {
            value: "Pacific/Auckland",
            viewValue: "UTC+12 - New Zealand Standard Time (NZST)",
        },
        {
            value: "Pacific/Fiji",
            viewValue: "UTC+12 - Fiji Standard Time (FJT)",
        },
        {
            value: "Pacific/Tongatapu",
            viewValue: "UTC+13 - Tonga Standard Time (TOT)",
        },
    ];

    const localTimeZone = Intl.DateTimeFormat().resolvedOptions().timeZone;
    let timeZoneValue: string = localTimeZone;
    let previewLocalValue: boolean = false;

    let timeTable: number[] = [];
    let timeTableInitiallySet: boolean = false;

    function onTimeZoneChange(event: Event) {
        timeZoneValue = (event.target! as HTMLSelectElement).value;
        timeZone.set(timeZoneValue);
        updateURL();
    }

    function handleTimeTableUpdate(event: CustomEvent) {
        timeTable = event.detail.timeTable;
        updateURL();
    }

    function updateURL() {
        if (!browser) return;
        const urlParams = new URLSearchParams(window.location.search);
        urlParams.set("zone", timeZoneValue);
        urlParams.set("table", encodeRanges(timeTable));
        const newUrl = `${window.location.origin}${
            window.location.pathname
        }?${urlParams.toString()}`;
        window.history.replaceState({}, "", newUrl);
    }

    function copyURL() {
        if (!browser) return;
        navigator.clipboard.writeText(window.location.href);
    }

    function onLocalClick() {
        previewLocalValue = true;
        previewLocal.set(true);
    }

    function onOriginalClick() {
        previewLocalValue = false;
        previewLocal.set(false);
    }

    onMount(() => {
        if (!browser) return;
        const urlParams = new URLSearchParams(window.location.search);
        const urlTimeZone = urlParams.get("zone");
        if (urlTimeZone) {
            timeZoneValue = urlTimeZone;
            timeZone.set(urlTimeZone);
        } else {
            timeZoneValue = localTimeZone;
            timeZone.set(localTimeZone);
        }
        const urlTimeTable = urlParams.get("table");
        if (urlTimeTable) {
            timeTable = decodeRanges(urlTimeTable);
            if (timeTable.length) {
                timeTableInitiallySet = true;
            }
        }
    });

    function encodeRanges(ids: number[]): string {
        if (!ids.length) return "";

        const sortedIds = [...ids].sort((a, b) => a - b);
        let [start, end] = [sortedIds[0], sortedIds[0]];
        const ranges = [];

        for (let id of sortedIds.slice(1)) {
            if (id !== end + 1) {
                ranges.push(start === end ? `${start}` : `${start}-${end}`);
                [start, end] = [id, id];
            } else {
                end = id;
            }
        }

        ranges.push(start === end ? `${start}` : `${start}-${end}`);
        return ranges.join(",");
    }

    function decodeRanges(rangesStr: string): number[] {
        return rangesStr.split(",").flatMap((range) => {
            const [start, end] = range.split("-").map(Number);

            return end === undefined || isNaN(start)
                ? [start]
                : Array.from({ length: end - start + 1 }, (_, i) => start + i);
        });
    }
</script>

<main>
    <h1>Availability Scheduler</h1>
    <span>Pick a time zone and schedule that works for you.</span>
    <span>When you're done, click "Copy URL" and return to the form.</span>
    <Input type="select" id="timeZone" bind:value={timeZoneValue} on:change={onTimeZoneChange}>
        {#each timezones as tz}
            <option value={tz.value}>{tz.viewValue}</option>
        {/each}
    </Input>
    {#if timeTableInitiallySet}
        <div class="d-flex flex-row justify-content-around">
            <Button color="primary" disabled={previewLocalValue} on:click={onLocalClick}>Local</Button>
            <b class="lh-lg">{previewLocalValue ? 'Local' : 'Original'}</b>
            <Button color="primary" disabled={!previewLocalValue} on:click={onOriginalClick}>Original</Button>
        </div>
    {/if}
    <TimeTable bind:timeTable on:message={handleTimeTableUpdate} />
    <Button color="primary" on:click={copyURL}>Copy URL</Button>
</main>

<style>
    main {
        padding: 1rem;
        width: 30rem;
        display: flex;
        flex-direction: column;
        gap: 1rem;
    }

    h1 {
        margin: 0;
        padding: 1rem 0;
    }
</style>
