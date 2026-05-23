<script>
  let liveRate = $state(null); 
  let isLoading = $state(true);
  let lastUpdated = $state(''); 
  
  let email = $state('');
  let subscribed = $state(false);
  let isSubmitting = $state(false);
  
  let errorMessage = $state('');
  let percentageProgress = $derived(liveRate ? ((liveRate / 100) * 100).toFixed(2) : "0.00");

  function formatDateTime(dateStr) {
    const dateObj = new Date(dateStr);
    const formattedDate = dateObj.toLocaleDateString('en-IN', {
      day: '2-digit',
      month: 'short',
      year: 'numeric'
    });
    const formattedTime = dateObj.toLocaleTimeString('en-IN', {
      hour: '2-digit',
      minute: '2-digit',
      hour12: true
    }).toUpperCase();
    return `${formattedDate}, ${formattedTime}`;
  }

  async function updateExchangeRate() {
    try {
      const res = await fetch('https://www.backend.arewe100yet.in/rate');
      if (!res.ok) throw new Error('Primary backend offline');
      const data = await res.json();
      
      if (data && data.USD_TO_INR) {
        liveRate = parseFloat(data.USD_TO_INR);
        // Sync timestamp extraction on primary channel
        if (data.time_last_update_utc) {
          lastUpdated = formatDateTime(data.time_last_update_utc);
        } else {
          lastUpdated = 'Today';
        }
        history[history.length - 1].rate = liveRate;
        isLoading = false; 
        return;
      }
      throw new Error('Primary backend returned unparseable shape');
    } catch (primaryErr) {
      console.warn('Primary telemetry failed, rolling back to fallback matrix:', primaryErr.message);
      try {
        const fallbackRes = await fetch('https://open.er-api.com/v6/latest/USD');
        if (!fallbackRes.ok) throw new Error('Fallback API network failure');
        
        const fallbackData = await fallbackRes.json();
        if (fallbackData.rates && fallbackData.rates.INR) {
          liveRate = parseFloat(fallbackData.rates.INR);

          if (fallbackData.time_last_update_utc) {
            lastUpdated = formatDateTime(fallbackData.time_last_update_utc);
          } else {
            lastUpdated = 'Today';
          }

          history[history.length - 1].rate = liveRate;
          isLoading = false; 
        } else {
          throw new Error('Fallback API structural failure');
        }
      } catch (fallbackErr) {
        console.error('CRITICAL: All exchange networks unresponsive.', fallbackErr);
        isLoading = false;
      }
    }
  }

  $effect(() => {
    updateExchangeRate();
    const interval = setInterval(updateExchangeRate, 300000);
    return () => clearInterval(interval);
  });
  
  let history = $state([
    { date: 'Jan 2026', rate: 89.96 },
    { date: 'Feb 2026', rate: 91.68 },
    { date: 'Mar 2026', rate: 93.48 },
    { date: 'Apr 2026', rate: 94.64 },
    { date: "Today" , rate: 96.9 }
  ]);
  
  const milestones = $derived([
    { value: '45.14', label: '2010', isLive: false, isTarget: false },
    { value: '64.15', label: '2015', isLive: false, isTarget: false },
    { value: '74.13', label: '2020', isLive: false, isTarget: false },
    { value: `${liveRate.toFixed(2)}`, label: '2026', isLive: true, isTarget: false },
    { value: '100.0', label: 'THIS YEAR??', isLive: false, isTarget: true }
  ]);

  async function handleSubscribe(e) {
    e.preventDefault();
    if (!email) return;
    isSubmitting = true;
    errorMessage = '';
    try {
      const response = await fetch('https://formspree.io/f/xwvzlobr', {
        method: 'POST',
        headers: {
          'Content-Type': 'application/json',
          'Accept': 'application/json'
        },
        body: JSON.stringify({ email: email })
      });
      if (response.ok) {
        subscribed = true;
        email = '';
        setTimeout(() => { subscribed = false; }, 5000);
      } else {
        const data = await response.json();
        errorMessage = data.errors ? data.errors.map(err => err.message).join(', ') : 'Something went wrong. Please try again.';
      }
    } catch (err) {
      errorMessage = "Oops! We encountered a small hiccup on our side. Please try again after some time.";
    } finally {
      isSubmitting = false;
    }
  }

  const minRate = 88;
  const maxRate = 100;
  function getSvgY(rate) {
    return 180 - ((rate - minRate) / (maxRate - minRate)) * 140;
  }
  
  let pointsStr = $derived(
    history.map((h, i) => `${(i * (600 / (history.length - 1)) + 20)},${getSvgY(h.rate)}`).join(' ')
  );
</script>

<svelte:head>
  <title>Are We 100 Yet? | The Rupee Tracker</title>
  <meta name="description" content="Track the historic drop of the Indian Rupee against the US Dollar with our simple progress tracker, milestone logs, and trend charts." />
</svelte:head>

{#if isLoading}
  <main class="w-full min-h-screen font-mono text-black bg-[#f3f3f3] flex items-center justify-center p-6">
    <div class="border-4 border-black bg-white p-8 shadow-[8px_8px_0px_0px_rgba(0,0,0,1)] text-center max-w-sm w-full">
      <div class="w-4 h-4 bg-orange-500 animate-ping mx-auto mb-4"></div>
      <p class="font-black text-xs tracking-widest uppercase">LOADING DATA...</p>
    </div>
  </main>
{:else}
  <main class="w-full min-h-screen font-mono text-black bg-[#f3f3f3] overflow-x-hidden">

  <!-- HERO BAND 1: THE LIVE FEED HEADER & VALUE INDEX -->
    <section class="w-full min-h-screen flex flex-col justify-between p-6 sm:p-10 md:p-16 box-border relative overflow-hidden bg-[#f7f7f7] border-b-[10px] border-black">
      <!-- Clean Structural Grid Blueprint Lines -->
      <div class="absolute inset-0 z-0 pointer-events-none opacity-[0.03]" style="background-image: linear-gradient(to right, black 1px, transparent 1px), linear-gradient(to bottom, black 1px, transparent 1px); background-size: 36px 36px;"></div>

      <header class="w-full flex flex-col md:flex-row justify-between items-start md:items-center gap-6 z-10">
        <div>
          <h1 class="text-5xl sm:text-7xl font-black uppercase tracking-tighter m-0 leading-none">
            AREWE<span class="text-orange-500">100</span>YET<span class="text-neutral-400 font-normal text-2xl">.IN</span>
          </h1>
          <p class="text-xs sm:text-sm font-sans font-bold text-neutral-500 mt-1.5 tracking-wide">A countdown to 100 Rupees per 1 Dollar.</p>
        </div>
      </header>

      <div class="my-auto z-10 py-12">
        <div>
          <span class="text-xs sm:text-sm font-black text-orange-600 tracking-widest uppercase bg-orange-50 px-2.5 py-1 border-2 border-orange-300 inline-block">// EXCHANGE RATE</span>
          <div class="flex flex-wrap items-baseline gap-2 sm:gap-6 mt-4">
            <span class="text-8xl sm:text-9xl md:text-[13rem] font-black tracking-tighter leading-none text-black">
              ₹{liveRate.toFixed(2)}
            </span>
            <span class="text-4xl sm:text-6xl font-light text-neutral-400">/ $1 USD</span>
          </div>
        </div>
      </div>

      <!-- CONTAINER GROUPED TO KEEP TIMESTAMP IMMEDIATELY ABOVE THE BLACK BAR -->
      <div class="w-full max-w-5xl z-10">
        <!-- TIMESTAMP SITTING DIRECTLY ABOVE THE BORDER -->
        <div class="flex flex-wrap items-center gap-x-3 gap-y-1 text-neutral-500 font-bold text-xs tracking-wide mb-3">
          {#if lastUpdated}
            <div class="flex items-center gap-2">
              <span class="inline-block w-2 h-2 bg-neutral-400 rounded-full animate-pulse"></span>
              <span class="text-neutral-400 font-medium">Last updated: <span class="font-bold text-neutral-600">{lastUpdated}</span></span>
            </div>
          {/if}
        </div>

        <!-- THE BLACK BAR & PARAGRAPH INFO -->
        <div class="border-t-4 border-black pt-8 flex flex-col sm:flex-row justify-between items-start sm:items-center gap-6">
          <p class="text-sm sm:text-base font-sans font-semibold text-neutral-700 max-w-lg leading-relaxed">
            The Indian Rupee is hitting historic lows. After starting the year at ₹90 per USD in January, it is sliding quickly toward the ₹100 mark. This page tracks the countdown to 100 Rupees per US Dollar.
          </p>
        </div>
      </div>
    </section>

    
    <!-- HERO BAND 2 -->
    <section class="w-full min-h-screen flex flex-col justify-between p-6 sm:p-10 md:p-16 box-border bg-white border-b-[10px] border-black relative">
      <div class="text-xs font-black text-neutral-400 tracking-widest uppercase">// THE CURRENT SCORE</div>
      
      <div class="my-auto space-y-10 max-w-6xl w-full mx-auto py-12">
        <div class="flex flex-col sm:flex-row justify-between items-start sm:items-end gap-6">
          <div>
            <h2 class="text-4xl sm:text-6xl font-black uppercase tracking-tighter">THE 100 BARRIER</h2>
            <p class="text-sm sm:text-base font-sans text-neutral-500 font-bold mt-2">Tracking how close we are to 100%.</p>
          </div>
          <span class="text-xl sm:text-2xl font-black bg-orange-500 text-white border-4 border-black px-6 py-3 shadow-[6px_6px_0px_0px_rgba(0,0,0,1)] shrink-0 tracking-tight">
            {percentageProgress}% PROGRESS SO FAR
          </span>
        </div>

        <div class="w-full bg-neutral-100 border-4 border-black h-24 sm:h-36 p-2 relative shadow-[10px_10px_0px_0px_rgba(0,0,0,1)]">
          <div class="bg-orange-500 h-full transition-all duration-1000 ease-out relative overflow-hidden" style="width: {percentageProgress}%;">
            <div class="absolute inset-0 bg-stripes opacity-[0.15]"></div>
          </div>
        </div>

        <div class="border-4 border-black p-5 sm:p-8 bg-amber-50 shadow-[6px_6px_0px_0px_rgba(0,0,0,1)] max-w-2xl">
          <span class="text-xs font-black block text-orange-600 tracking-wider mb-1">THE FINAL GAP</span>
          <p class="text-lg sm:text-2xl font-black leading-tight">
           We are just <span class="bg-black text-white px-2.5 py-0.5 font-black">₹{(100 - liveRate).toFixed(2)}</span> away from hitting a century. 
          </p>
        </div>
      </div>
      <div></div>
    </section>

<!-- HERO BAND 3: CHART TIMELINE DISPLAY -->
  <section class="w-full min-h-screen flex flex-col justify-between p-6 sm:p-10 md:p-16 box-border bg-[#fdfdfd] border-b-[10px] border-black">
    <div class="text-xs font-black text-neutral-400 tracking-widest uppercase">// PAST RATES</div>

    <div class="my-auto max-w-6xl w-full mx-auto space-y-8 py-12">
      <div class="border-b-4 border-black pb-4">
        <h2 class="text-3xl sm:text-5xl font-black uppercase tracking-tighter flex items-center gap-4">
          <span class="w-5 h-5 bg-orange-500 inline-block"></span> THE RUPEE OVER TIME
        </h2>
        <p class="text-xs sm:text-sm font-sans font-bold text-neutral-500 mt-1 tracking-wide">Tracking the daily changes over time.</p>
      </div>

      <div class="w-full overflow-x-auto pt-8 pb-4">
        <!-- FIXED: Increased viewBox width to 660 to give right side text its own spacious margin -->
        <svg viewBox="0 0 660 235" class="w-full min-w-[660px] h-auto block overflow-visible">
          <defs>
            <!-- Mask boundary matches your original line span perfectly -->
            <clipPath id="chart-boundary">
              <rect x="35" y="0" width="570" height="235" />
            </clipPath>
          </defs>

          <!-- TARGET GUIDE RAILS -->
          <line x1="35" y1="20" x2="605" y2="20" stroke="#000" stroke-width="2.5" stroke-dasharray="4 4" class="opacity-[0.15]" />
          <!-- FIXED: Pushed x out to 620 to add distinct padding between the line edge and ceiling text -->
          <text x="620" y="24" class="text-[11px] font-black fill-orange-600" text-anchor="start">100.0</text>
          
          <line x1="35" y1="90" x2="605" y2="90" stroke="#000" stroke-width="2.5" stroke-dasharray="4 4" class="opacity-[0.15]" />
          <text x="620" y="94" class="text-[11px] font-black fill-neutral-400" text-anchor="start">94.0</text>
          
          <line x1="35" y1="160" x2="605" y2="160" stroke="#000" stroke-width="2.5" stroke-dasharray="4 4" class="opacity-[0.15]" />
          <text x="620" y="164" class="text-[11px] font-black fill-neutral-400" text-anchor="start">88.0</text>

          <!-- MAIN POLYLINE GRAPH TRACK -->
          <g clip-path="url(#chart-boundary)">
            <polyline fill="none" stroke="#f97316" stroke-width="6" stroke-linecap="square" points={pointsStr} />
          </g>

          {#each history as point, i}
            <!-- RESTORED: Your original clean mathematical point positioning -->
            {@const x = (i * (570 / (history.length - 1)) + 35)}
            {@const y = getSvgY(point.rate)}
            
            <circle cx={x} cy={y} r="8" class="fill-white stroke-black stroke-2" />
            <text x={x} y={y - 18} class="text-xs font-black fill-black" text-anchor="middle">₹{point.rate.toFixed(2)}</text>
            
            <!-- FIXED: Shifted bottom date rendering context down to y="222" for vertical padding from the plot area -->
            <text x={x} y="222" class="text-[10px] uppercase font-black fill-neutral-400" text-anchor="middle">{point.date}</text>
          {/each}
        </svg>
      </div>
    </div>

    <div class="text-xs font-sans text-neutral-400 font-bold">Numbers update once every day.</div>
  </section>

    <!-- HERO BAND 4 -->
    <section class="w-full min-h-screen flex flex-col justify-between p-6 sm:p-10 md:p-16 box-border bg-black text-white relative">
      <div class="text-xs font-black text-neutral-500 tracking-widest uppercase">// WHAT NOW ?</div>

      <div class="my-auto max-w-6xl w-full mx-auto grid grid-cols-1 lg:grid-cols-2 gap-12 xl:gap-24 items-center py-12">
        <div class="space-y-6">
          <h3 class="text-xs font-black uppercase tracking-widest text-neutral-400">// THE RUPEE'S MILESTONES</h3>
          <div class="space-y-3">
            {#each milestones as m}
              <div class="flex items-center gap-4">
                <div class="w-10 h-10 text-xs flex items-center justify-center font-black border-2 shrink-0
                  {m.isLive ? 'bg-orange-500 text-white border-white scale-105 animate-pulse' : 'bg-neutral-900 text-neutral-500 border-neutral-800'}"
                >
                  {m.isLive ? '●' : '✓'}
                </div>
                <div class="flex-1 flex justify-between items-center px-5 py-3 border-2 transition-all
                  {m.isLive ? 'bg-neutral-900 border-orange-500 shadow-[4px_4px_0px_0px_rgba(249,115,22,1)]' : 'bg-neutral-900/40 border-neutral-800'}"
                >
                  <span class="text-base font-black {m.isLive ? 'text-orange-500' : 'text-white'}">₹{m.value}</span>
                  <span class="text-[11px] font-black uppercase tracking-widest text-neutral-400">{m.label}</span>
                </div>
              </div>
            {/each}
          </div>
        </div>

        <div class="bg-white border-4 border-black p-6 sm:p-10 text-black shadow-[10px_10px_0px_0px_rgba(249,115,22,1)]">
          <h3 class="text-xl sm:text-2xl font-black uppercase tracking-tight border-b-4 border-black pb-3 mb-4">[GET EMAIL UPDATES]</h3>
          <p class="text-xs sm:text-sm text-neutral-600 font-sans font-semibold leading-relaxed mb-6">
            Sign up below to get an alert when the Rupee gets closer to 100, so you can see the chart hit triple digits live.
          </p>

          <form onsubmit={handleSubscribe} class="space-y-4">
            <input id="email" type="email" required bind:value={email} disabled={isSubmitting} placeholder="operator@network.com" class="w-full bg-neutral-50 border-4 border-black p-4 text-xs sm:text-sm font-bold focus:outline-none focus:border-orange-500 placeholder:text-neutral-300 rounded-none" />
            <button type="submit" disabled={isSubmitting} class="w-full bg-black hover:bg-orange-500 text-white text-xs sm:text-sm font-black uppercase py-4.5 tracking-widest border-4 border-black shadow-[4px_4px_0px_0px_rgba(0,0,0,0.15)] transition-colors rounded-none">
              {isSubmitting ? 'TRANSMITTING ROUTE...' : 'KEEP ME UPDATED'}
            </button>
          </form>

          {#if subscribed}
            <div class="mt-4 p-2.5 bg-black text-orange-400 text-xs font-black uppercase text-center tracking-widest animate-pulse">
              ✓ ENDPOINT SYSTEM RECOGNIZED.
            </div>
          {/if}
          {#if errorMessage}
            <div class="mt-4 p-2.5 bg-red-600 text-white text-xs font-black uppercase text-center tracking-wider">
              ⚠️ {errorMessage}
            </div>
          {/if}
        </div>
      </div>

      <footer class="w-full border-t-2 border-neutral-800 pt-6 flex flex-col md:flex-row gap-6 justify-between items-center text-[10px] font-bold text-neutral-500">
        <p class="text-center md:text-left tracking-tight">
          AREWE100YET.IN · INDEPENDENT SATIRICAL MACRO OVERVIEW · NOT ACTIONABLE TRADING INTELLIGENCE.
        </p>
        <div class="bg-yellow-950/50 border border-yellow-600/20 text-yellow-500/70 px-4 py-1.5 text-[9px] max-w-md text-center md:text-right">
          WARNING: This is just a tracker. Do not use this page to trade or invest money.
        </div>
      </footer>
    </section>
  </main>
{/if}

<style>
  .bg-stripes {
    background-image: linear-gradient(45deg, #000 25%, transparent 25%, transparent 50%, #000 50%, #000 75%, transparent 75%, transparent);
    background-size: 16px 16px;
  }
</style>
