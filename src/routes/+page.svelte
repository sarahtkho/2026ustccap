<script>
  import { onMount, onDestroy } from 'svelte';
  import { Html5Qrcode } from 'html5-qrcode';

  let scanner;
  let scanning = false;
  let result = null;
  let groupCheckin = false;
  let loading = false;
  let error = '';

  // Replace with your API endpoint
  const API_URL = 'https://your-api.com/checkin';

  onMount(() => {
    scanner = new Html5Qrcode('reader');
  });

  onDestroy(() => {
    if (scanner) scanner.clear();
  });

  async function startScan() {
    try {
      await scanner.start(
        { facingMode: 'environment' },
        { fps: 10, qrbox: { width: 250, height: 250 } },
        onScanSuccess,
        (err) => console.warn('Scan error:', err)
      );
      scanning = true;
    } catch (err) {
      error = 'Camera access denied or unavailable';
    }
  }

  async function stopScan() {
    await scanner.stop();
    scanning = false;
  }

  async function onScanSuccess(decodedText) {
    try {
      const response = await fetch(`${API_URL}?qr=${encodeURIComponent(decodedText)}`);
      result = await response.json();
      stopScan();
    } catch (err) {
      error = 'API fetch failed';
    }
  }

  async function handleCheckin() {
    loading = true;
    try {
      const payload = {
        qrData: result,
        checkAllGuests: groupCheckin
      };
      await fetch(API_URL, {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify(payload)
      });
      result = null;
      groupCheckin = false;
      error = 'Check-in successful!';
      startScan(); // Restart scanner
    } catch (err) {
      error = 'Check-in failed';
    }
    loading = false;
  }
</script>

<svelte:head>
  <title>Concert QR Check-in</title>
</svelte:head>

<div class="container">
  {#if !result}
    <div id="reader" class="scanner"></div>
    {#if scanning}
      <button onclick={stopScan}>Stop Scan</button>
    {:else}
      <button onclick={startScan}>Start Scan</button>
    {/if}
  {:else}
    <h2>Scan Result</h2>
    <pre>{JSON.stringify(result, null, 2)}</pre>
    {#if result.group && result.guests}
      <label>
        <input type="checkbox" bind:checked={groupCheckin} />
        Check in all guests in group "{result.group}"
      </label>
    {/if}
    <button onclick={handleCheckin} disabled={loading}>
      {loading ? 'Processing...' : 'Confirm Check-in'}
    </button>
    <button onclick={() => { result = null; startScan(); }}>Rescan</button>
  {/if}

  {#if error}
    <p class="error">{error}</p>
  {/if}
</div>

<style>
  .container {
    max-width: 600px;
    margin: 0 auto;
    padding: 20px;
    font-family: system-ui;
  }
  .scanner {
    width: 100%;
    height: 400px;
    border: 2px solid #ccc;
    border-radius: 8px;
  }
  pre {
    background: #f4f4f4;
    padding: 16px;
    border-radius: 4px;
    overflow: auto;
  }
  button {
    padding: 12px 24px;
    margin: 8px;
    background: #007acc;
    color: white;
    border: none;
    border-radius: 4px;
    cursor: pointer;
  }
  button:disabled {
    background: #ccc;
  }
  .error {
    color: red;
  }
</style>
