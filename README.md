const express = require('express');
const cors = require('cors');
const nodemailer = require('nodemailer');

const PORT = process.env.PORT || 5000;
const app = express();
app.use(express.json());
app.use(cors());

const ADMIN_EMAIL = 'your@email.com'; // <-- Where the reports go

const DIMENSIONS = [
  "Role Definition Integrity",
  "Operational Infrastructure",
  "Leadership Capacity",
  "Decision Authority Distribution",
  "Financial Readiness"
];

function interpretScore(value) {
  switch (Number(value)) {
    case 1: return "Critical fragility";
    case 2: return "Weak structure, high risk";
    case 3: return "Developing, some resilience";
    case 4: return "Strong, but could improve";
    case 5: return "Exceptional resilience";
    default: return "No score";
  }
}

// Free SMTP example: Gmail. Use app password for extra security.
const transporter = nodemailer.createTransport({
  service: 'gmail',
  auth: {
    user: 'your-smtp-address@gmail.com', // Replace with sender
    pass: 'your-app-password' // Use App Passwords for Gmail
  }
});

app.post('/api/submit', async (req, res) => {
  try {
    const { name, email, company, position, answers } = req.body;

    // Score and analyze
    let body = `
      <h2>Operational Intelligence Diagnostic Submission</h2>
      <b>From:</b> ${name} (${position})<br>
      <b>Company:</b> ${company}<br>
      <b>Contact:</b> ${email}<br><br>
      <hr><br>
    `;

    let fullScore = 0;
    DIMENSIONS.forEach((dim, dIdx) => {
      const sectionScores = answers[dIdx].map(a => Number(a));
      const avg = sectionScores.reduce((s,v)=>s+v,0) / sectionScores.length;
      fullScore += avg;
      body += `<b>${dim}:</b> <br>`;
      answers[dIdx].forEach((ans, qIdx) => {
        body += `Q${qIdx+1}: Score ${ans} – <i>${interpretScore(ans)}</i><br>`;
      });
      body += `<b>→ Avg for Dimension:</b> ${avg.toFixed(2)}<br><br>`;
    });
    body += `<hr>`;
    body += `<h3>Total Average Score: ${(fullScore/DIMENSIONS.length).toFixed(2)} / 5</h3>`;
    // Add premium-level, dimension-specific commentary
    body += analyzeStructure(answers);

    // Send to admin only
    await transporter.sendMail({
      from: '"OpIntel Diagnostic" <your-smtp-address@gmail.com>',
      to: ADMIN_EMAIL,
      subject: `New Diagnostic Submission: ${company} / ${name}`,
      html: body,
    });

    res.json({ ok: true });
  } catch (err) {
    console.error(err);
    res.status(500).json({ ok: false, error: "Failed to send report." });
  }
});

// Algos for premium commentary (replace with your own!)
function analyzeStructure(answers) {
  let out = "<h3>Structural Analysis:</h3>";
  if (answers[0].some(ans => ans <= 2)) {
    out +=
      "<b>Role Definition Integrity:</b> Role ambiguity may be causing significant delivery risk. Recommend substantial review and redesign of internal accountabilities.<br>";
  }
  if (answers[2].some(ans => ans <= 2)) {
    out +=
      "<b>Leadership Capacity:</b> Leadership gaps detected—likely to impair threat mitigation and oversight. Suggest strengthening cross-functional accountability.<br>";
  }
  // Extend with further dimension-specific advice...
  return out;
}

app.listen(PORT, () => {
  console.log(`Backend running on port ${PORT}`);
});
