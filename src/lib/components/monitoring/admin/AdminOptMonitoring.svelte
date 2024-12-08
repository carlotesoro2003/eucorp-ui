<script lang="ts">
    import { supabase } from "$lib/supabaseClient";
    import { onMount } from "svelte";
    import jsPDF from "jspdf";
    import autoTable from "jspdf-autotable";

    interface Opportunity {
        id: number;
        opt_statement: string;
        kpi: string;
        planned_actions: string;
        evaluation: string | null;
        achieved: boolean;
        time_completed: string | null;
    }

    let opportunities: Opportunity[] = [];
    let filteredOpportunities: Opportunity[] = [];
    let filterStatus: "all" | "achieved" | "not_achieved" = "all";
    let isLoading = true;
    let summaryReport = ""; // Holds the generated summary report
    let isGeneratingSummary = false;

    // Fetch opportunities
    const fetchOpportunities = async () => {
    isLoading = true;

    try {
        // Fetch data from Supabase
        const { data, error } = await supabase
            .from("opt_monitoring")
            .select(`
                opt_id,
                opportunities (
                    opt_statement,
                    kpi,
                    planned_actions
                ),
                evaluation,
                is_accomplished,
                time_completed
            `);

        // Handle errors
        if (error) {
            console.error("Error fetching opportunities:", error);
            return;
        }

        // Process and map data
        if (data) {
            opportunities = data.map((item: any) => ({
                id: item.opt_id,
                opt_statement: item.opportunities?.opt_statement || "No Statement Available",
                kpi: item.opportunities?.kpi || "No KPI Available",
                planned_actions: item.opportunities?.planned_actions || "No Actions Available",
                evaluation: item.evaluation || "Pending Evaluation",
                achieved: item.is_accomplished || false,
                time_completed: item.time_completed || null,
            }));

            // Apply the current filter
            applyFilter();
        }
    } catch (error) {
        // Catch unexpected errors
        console.error("Unexpected error during data fetch:", error);
    } finally {
        // Ensure the loading state is cleared
        isLoading = false;
    }
};


    // Apply filter based on status
    const applyFilter = () => {
        if (filterStatus === "all") {
            filteredOpportunities = opportunities;
        } else if (filterStatus === "achieved") {
            filteredOpportunities = opportunities.filter((o) => o.achieved);
        } else if (filterStatus === "not_achieved") {
            filteredOpportunities = opportunities.filter((o) => !o.achieved);
        }
    };

    // Generate a PDF report with the summary
    const generateSummaryPDF = async () => {
        isGeneratingSummary = true;

        try {
            const response = await fetch("/api/summary-report-opt-monitoring", {
                method: "POST",
                headers: { "Content-Type": "application/json" },
                body: JSON.stringify({ opportunities: filteredOpportunities }),
            });

            const data = await response.json();

            if (!response.ok || data.error) {
                throw new Error(data.error || "Failed to generate summary.");
            }

            // Create the PDF with the summary report
            const doc = new jsPDF();
            const title = `Summary Report`;

            doc.setFont("times", "normal");

            // Title and Header Information
            doc.setFontSize(12);
            doc.text("MANUEL S. ENVERGA UNIVERSITY FOUNDATION", 14, 10);
            doc.setFontSize(10);
            doc.text("SY 2024-2025", 14, 20);

            doc.setFontSize(14);
            doc.text(title, 14, 15);

            const summaryLines = doc.splitTextToSize(data.summaryReport, 180);
            doc.setFontSize(12);
            doc.text(summaryLines, 14, 35);

            doc.save("OpportunitiesSummary.pdf");
        } catch (error) {
            console.error("Error generating summary report:", error);
            alert("An error occurred while generating the summary report.");
        } finally {
            isGeneratingSummary = false;
        }
    };

    onMount(fetchOpportunities);
</script>

<!-- HTML for Opportunities Monitoring -->
<div class="min-h-screen p-8">
    <h1 class="text-3xl font-bold mb-6">Opportunities Monitoring</h1>

    <div class="flex space-x-4 mb-4">
        <button class="btn btn-primary" on:click={() => { filterStatus = "all"; applyFilter(); }}>Show All</button>
        <button class="btn btn-success" on:click={() => { filterStatus = "achieved"; applyFilter(); }}>Show Achieved</button>
        <button class="btn btn-error" on:click={() => { filterStatus = "not_achieved"; applyFilter(); }}>Show Not Achieved</button>
        <button class="btn btn-accent ml-auto" on:click={generateSummaryPDF} disabled={isGeneratingSummary}>
            {isGeneratingSummary ? "Generating..." : "Generate Summary Report"}
        </button>
    </div>

    {#if isLoading}
        <div class="text-center text-xl">Loading...</div>
    {:else if filteredOpportunities.length > 0}
        <table class="table-auto w-full text-sm">
            <thead>
                <tr>
                    <th>Opportunity Statement</th>
                    <th>KPI</th>
                    <th>Planned Actions</th>
                    <th>Evaluation</th>
                    <th>Status</th>
                    <th>Time Completed</th>
                </tr>
            </thead>
            <tbody>
                {#each filteredOpportunities as o}
                    <tr>
                        <td>{o.opt_statement}</td>
                        <td>{o.kpi}</td>
                        <td>{o.planned_actions}</td>
                        <td>{o.evaluation || "Pending Evaluation"}</td>
                        <td>{o.achieved ? "Achieved" : "Not Achieved"}</td>
                        <td>{o.time_completed ? new Date(o.time_completed).toLocaleString() : "N/A"}</td>
                    </tr>
                {/each}
            </tbody>
        </table>
    {:else}
        <div>No opportunities found for this filter.</div>
    {/if}
</div>

<style>
    .btn {
        padding: 8px 12px;
        border-radius: 8px;
        cursor: pointer;
    }
</style>
