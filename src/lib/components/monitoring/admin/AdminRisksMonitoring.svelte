<script lang="ts">
  import { supabase } from "$lib/supabaseClient";
  import { onMount } from "svelte";

  interface RiskMonitoring {
    id: number;
    rrn: string;
    risk_statement: string;
    likelihood_rating: string | null;
    severity: string | null;
    control_rating: string | null;
    monitoring_rating: string | null;
    is_achieved: boolean;
  }

  let risksMonitoring: RiskMonitoring[] = [];
  let isLoading = true;
  let errorMessage: string | null = null;

  // Fetch risk monitoring data
  const fetchRiskMonitoring = async () => {
    isLoading = true;
    try {
      const { data, error } = await supabase.from("risk_monitoring").select(`
                id,
                risks (
                    rrn,
                    risk_statement
                ),
                likelihood_rating:likelihood_rating_id(name),
                severity:severity_id(name),
                control_rating:control_rating_id(name),
                monitoring_rating:monitoring_rating_id(status),
                is_achieved
            `);

      if (error) throw error;

      risksMonitoring = data.map((item: any) => ({
        id: item.id,
        rrn: item.risks.rrn,
        risk_statement: item.risks.risk_statement,
        likelihood_rating: item.likelihood_rating?.name || "Not Available",
        severity: item.severity?.name || "Not Available",
        control_rating: item.control_rating?.name || "Not Available",
        monitoring_rating: item.monitoring_rating?.status || "Not Available",
        is_achieved: item.is_achieved,
      }));
    } catch (error) {
      console.error("Error fetching risk monitoring data:", error);
      errorMessage = "Failed to fetch risk monitoring data.";
    } finally {
      isLoading = false;
    }
  };

  // Load data on mount
  onMount(() => {
    fetchRiskMonitoring();
  });
</script>

<div class="container mx-auto my-6">
  <h1 class="text-2xl font-bold mb-4">Risks Monitoring</h1>

  {#if isLoading}
    <div class="text-center text-xl">
      <span class="loading loading-spinner loading-md"></span>
    </div>
  {:else}
    {#if errorMessage}
      <div class="alert alert-error">{errorMessage}</div>
    {/if}

    <div class="overflow-x-auto">
      <table class="table w-full">
        <thead>
          <tr>
            <th>RRN</th>
            <th>Risk Statement</th>
            <th>Likelihood Rating</th>
            <th>Severity</th>
            <th>Control Rating</th>
            <th>Monitoring Rating</th>
            <th>Status</th>
          </tr>
        </thead>
        <tbody>
          {#each risksMonitoring as risk}
            <tr>
              <td>{risk.rrn}</td>
              <td>{risk.risk_statement}</td>
              <td>{risk.likelihood_rating}</td>
              <td>{risk.severity}</td>
              <td>{risk.control_rating}</td>
              <td>{risk.monitoring_rating}</td>
              <td>
                {risk.is_achieved ? "Achieved" : "Still mitigating the risk"}
              </td>
            </tr>
          {/each}
        </tbody>
      </table>
    </div>
  {/if}
</div>
