<script>
  import { onMount } from 'svelte';
  import Chart from 'chart.js/auto';
  import { fade } from 'svelte/transition';
  import 'chartjs-adapter-date-fns';
  import { enUS } from 'date-fns/locale';

  let ctx;
  let chart;
  let activeTab = 'income';
  let timeRange = 1; // Default to 1 year
  let incomeData = [];
  let expenseData = [];
  let newName = '';
  let newAmount = '';
  let newStartDate = '';
  let newEndDate = '';
  let newFrequency = 'monthly';

  const frequencies = ['daily', 'weekly', 'monthly', 'yearly', 'one-time'];

  function addEntry() {
    if (newName && newAmount && newStartDate && newFrequency) {
      const newEntry = {
        name: newName,
        amount: parseFloat(newAmount),
        startDate: new Date(newStartDate),
        endDate: newEndDate ? new Date(newEndDate) : null,
        frequency: newFrequency
      };
      if (activeTab === 'income') {
        incomeData = [...incomeData, newEntry];
      } else {
        expenseData = [...expenseData, newEntry];
      }
      newName = '';
      newAmount = '';
      newStartDate = '';
      newEndDate = '';
      newFrequency = 'monthly';
      updateChart();
    }
  }

  function deleteEntry(index) {
    if (activeTab === 'income') {
      incomeData = incomeData.filter((_, i) => i !== index);
    } else {
      expenseData = expenseData.filter((_, i) => i !== index);
    }
    updateChart();
  }

  function calculateCashFlow(startDate, endDate) {
    let cashFlow = 0;
    const dailyCashFlow = new Array(Math.ceil((endDate - startDate) / (1000 * 60 * 60 * 24))).fill(0);

    function addToCashFlow(entry, multiplier = 1) {
      const amount = entry.amount * multiplier;
      const start = Math.max(0, Math.floor((entry.startDate - startDate) / (1000 * 60 * 60 * 24)));
      const end = entry.endDate ? Math.floor((entry.endDate - startDate) / (1000 * 60 * 60 * 24)) : dailyCashFlow.length - 1;

      switch (entry.frequency) {
        case 'daily':
          for (let i = start; i <= end; i++) {
            dailyCashFlow[i] += amount;
          }
          break;
        case 'weekly':
          for (let i = start; i <= end; i += 7) {
            dailyCashFlow[i] += amount;
          }
          break;
        case 'monthly':
          for (let i = start; i <= end; i += 30) {
            dailyCashFlow[i] += amount;
          }
          break;
        case 'yearly':
          for (let i = start; i <= end; i += 365) {
            dailyCashFlow[i] += amount;
          }
          break;
        case 'one-time':
          dailyCashFlow[start] += amount;
          break;
      }
    }

    incomeData.forEach(entry => addToCashFlow(entry));
    expenseData.forEach(entry => addToCashFlow(entry, -1));

    return dailyCashFlow.reduce((acc, value) => {
      cashFlow += value;
      acc.push(cashFlow);
      return acc;
    }, []);
  }

  function updateChart() {
    if (!chart) return;

    const startDate = new Date();
    const endDate = new Date(startDate.getFullYear() + timeRange, startDate.getMonth(), startDate.getDate());
    const cashFlowData = calculateCashFlow(startDate, endDate);
    const labels = cashFlowData.map((_, index) => {
      const date = new Date(startDate.getTime() + index * 24 * 60 * 60 * 1000);
      return date.toISOString().split('T')[0];
    });

    chart.data.labels = labels;
    chart.data.datasets[0].data = cashFlowData;
    chart.options.scales.x.time.unit = timeRange > 2 ? 'month' : 'week';
    chart.update();
  }

  onMount(() => {
    Chart.defaults.locale = enUS;
    chart = new Chart(ctx, {
      type: 'line',
      data: {
        labels: [],
        datasets: [{
          label: 'Cash Balance',
          data: [],
          borderColor: 'rgb(75, 192, 192)',
          backgroundColor: 'rgba(75, 192, 192, 0.2)',
          tension: 0.1,
          fill: true
        }]
      },
      options: {
        responsive: true,
        plugins: {
          legend: {
            position: 'top',
          },
          title: {
            display: true,
            text: 'Cash Balance Over Time'
          }
        },
        scales: {
          x: {
            type: 'time',
            time: {
              unit: 'week'
            },
            adapters: {
              date: {
                locale: enUS
              }
            }
          },
          y: {
            beginAtZero: true
          }
        }
      }
    });

    updateChart();
  });
</script>

<svelte:head>
  <title>Financial Dashboard</title>
</svelte:head>

<div class="container mx-auto p-4 max-w-4xl pb-[125px]">
  <h1 class="text-3xl font-bold mb-6 text-center text-gray-800">Financial Dashboard</h1>

  <div class="mb-6 flex justify-center">
    <button
      class="px-6 py-2 rounded-l-lg {activeTab === 'income' ? 'bg-blue-500 text-white' : 'bg-gray-200 text-gray-700'} focus:outline-none transition duration-300"
      on:click={() => activeTab = 'income'}
    >
      Income
    </button>
    <button
      class="px-6 py-2 rounded-r-lg {activeTab === 'expenses' ? 'bg-blue-500 text-white' : 'bg-gray-200 text-gray-700'} focus:outline-none transition duration-300"
      on:click={() => activeTab = 'expenses'}
    >
      Expenses
    </button>
  </div>

  <div class="mb-6 flex justify-center items-center">
    <label for="timeRange" class="mr-2 text-gray-700">Time Range:</label>
    <select id="timeRange" bind:value={timeRange} on:change={updateChart} class="border rounded p-2 focus:outline-none focus:ring-2 focus:ring-blue-500">
      <option value={1}>1 Year</option>
      <option value={2}>2 Years</option>
      <option value={5}>5 Years</option>
      <option value={10}>10 Years</option>
    </select>
  </div>

  <div class="mb-8 bg-white p-4 rounded-lg shadow-md">
    <canvas bind:this={ctx}></canvas>
  </div>

  <div class="mb-8">
    <h2 class="text-2xl font-bold mb-4 text-gray-800">{activeTab === 'income' ? 'Income' : 'Expenses'}</h2>
    <div class="overflow-x-auto">
      <table class="w-full text-left border-collapse">
        <thead>
          <tr class="bg-gray-100">
            <th class="p-3 border">Name</th>
            <th class="p-3 border">Amount</th>
            <th class="p-3 border">Start Date</th>
            <th class="p-3 border">End Date</th>
            <th class="p-3 border">Frequency</th>
            <th class="p-3 border">Actions</th>
          </tr>
        </thead>
        <tbody>
          {#each activeTab === 'income' ? incomeData : expenseData as entry, i}
            <tr class="hover:bg-gray-50">
              <td class="p-3 border">{entry.name}</td>
              <td class="p-3 border">${entry.amount.toFixed(2)}</td>
              <td class="p-3 border">{entry.startDate.toISOString().split('T')[0]}</td>
              <td class="p-3 border">{entry.endDate ? entry.endDate.toISOString().split('T')[0] : 'N/A'}</td>
              <td class="p-3 border">{entry.frequency}</td>
              <td class="p-3 border">
                <button on:click={() => deleteEntry(i)} class="text-red-500 hover:text-red-700 focus:outline-none">Delete</button>
              </td>
            </tr>
          {/each}
        </tbody>
      </table>
    </div>
  </div>

  <div class="mb-8">
    <h3 class="text-xl font-bold mb-4 text-gray-800">Add New {activeTab === 'income' ? 'Income' : 'Expense'}</h3>
    <div class="grid grid-cols-1 md:grid-cols-2 lg:grid-cols-3 gap-4">
      <input bind:value={newName} placeholder="Name" class="border rounded p-2 w-full" />
      <input bind:value={newAmount} type="number" placeholder="Amount" class="border rounded p-2 w-full" />
      <input bind:value={newStartDate} type="date" placeholder="Start Date" class="border rounded p-2 w-full" />
      <input bind:value={newEndDate} type="date" placeholder="End Date" class="border rounded p-2 w-full" />
      <select bind:value={newFrequency} class="border rounded p-2 w-full">
        {#each frequencies as freq}
          <option value={freq}>{freq}</option>
        {/each}
      </select>
      <button on:click={addEntry} class="bg-green-500 text-white px-4 py-2 rounded hover:bg-green-600 transition duration-300">Add</button>
    </div>
  </div>
</div>