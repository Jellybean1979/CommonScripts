// ─── COMMIT HASH ─────────────────────────────────────
async function updateCommitHash() {
  const repo   = "jellybean1979/Hangman";
  const branch = "main";
  try {
    const response = await fetch(`https://api.github.com/repos/${repo}/commits/${branch}`);
    const data = await response.json();
    const hash = data.sha.substring(0, 7);
    document.getElementById("versionTag").textContent = `by Gary Langley — build ${hash}`;
  } catch (err) {
    document.getElementById("versionTag").textContent = "by Gary Langley — build unknown";
  }
}
