<!-- Embed this in a Markdown file using raw HTML blocks -->

<h2>🔍 HTTP Security Header Scanner</h2>
<p>Enter a URL to check for missing or misconfigured security headers:</p>

<input type="text" id="url" placeholder="https://example.com" size="40" />
<button onclick="checkHeaders()">Check</button>

<div id="result" style="margin-top: 1rem;"></div>

<script>
  async function checkHeaders() {
    const url = document.getElementById("url").value;
    const output = document.getElementById("result");
    output.innerHTML = "🔄 Checking headers...";

    try {
      const response = await fetch("https://api.allorigins.win/get?url=" + encodeURIComponent(url));
      const text = await response.json();
      const headers = response.headers;

      const importantHeaders = [
        "strict-transport-security",
        "content-security-policy",
        "x-frame-options",
        "x-content-type-options",
        "referrer-policy",
        "permissions-policy"
      ];

      let resultHTML = "<h3>✅ Results:</h3><ul>";
      for (const header of importantHeaders) {
        const value = headers.get(header);
        resultHTML += `<li><strong>${header}:</strong> ${value || "<span style='color:red;'>Missing</span>"}</li>`;
      }
      resultHTML += "</ul>";
      output.innerHTML = resultHTML;
    } catch (e) {
      output.innerHTML = "❌ Could not fetch headers. The site may block CORS or the proxy is down.";
    }
  }
</script>
