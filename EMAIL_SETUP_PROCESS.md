# Complete Process: Fix Greeting Email After Approval

## Problem

When an admin approves a student registration, the approval succeeds (status updated to "Approved" in localStorage), but the greeting email fails to send. The toast shows:

> "Approval completed, but the greeting email could not be sent."

## Root Cause

The `sendGreetingEmail()` function in `admin.js` passes template parameters (`name`, `email`) that do **not** match standard EmailJS template variable conventions. EmailJS requires specific parameter names — most critically `to_email` for the recipient's email address. If the EmailJS template's "To" field references `{{to_email}}` but the code sends `email`, EmailJS cannot resolve the recipient and the send fails.

## Complete Fix Process

### Step 1: Fix the `sendGreetingEmail` function in `admin.js`

Replace the existing function with one that uses standard EmailJS template parameter names:

```javascript
function sendGreetingEmail(studentName, studentEmail) {
  const templateParams = {
    to_name: studentName,
    to_email: studentEmail,
    from_name: 'InnoPitch & Greenovation Mavericks Club',
    subject: 'Welcome to InnoPitch — Your Application Has Been Approved!',
    message: `Dear ${studentName},

Congratulations! Your application to join InnoPitch & Greenovation Mavericks Club has been approved.

Welcome aboard! We look forward to seeing you at our upcoming events and activities.

Best regards,
The InnoPitch Team`
  };

  return emailjs.send(
    'service_3s6k61q',
    'template_z7dg3ls',
    templateParams
  );
}
```

**Key changes:**
| Old Parameter | New Parameter | Why |
|---|---|---|
| `name` | `to_name` | Standard EmailJS recipient name variable |
| `email` | `to_email` | Standard EmailJS recipient email variable (required for delivery) |
| *(missing)* | `from_name` | Sender name shown in the email |
| *(missing)* | `subject` | Email subject line |
| *(missing)* | `message` | Email body content |

### Step 2: Improve error handling in the approve handler

Update the approve action handler to surface the actual EmailJS error message:

```javascript
if (action === 'approve') {
  await postAdminAction('update', id, { Status: 'Approved' });
  try {
    await sendGreetingEmail(row.Name, row.Email);
    showToast('Student approved and greeting email sent successfully.', 'success');
  } catch (emailError) {
    console.error('Greeting email error:', emailError);
    const detail = (emailError && (emailError.text || emailError.message)) || 'Unknown error';
    showToast(`Approval completed, but the greeting email could not be sent. (${detail})`, 'error');
  }
}
```

### Step 3: Configure the EmailJS template

1. Go to [https://dashboard.emailjs.com](https://dashboard.emailjs.com) and sign in.
2. Navigate to **Email Templates** → select the template with ID `template_z7dg3ls` (or create a new one).
3. In the template editor, set the following fields:

   **To (Email Address):** `{{to_email}}`

   **Subject:** `{{subject}}`

   **From Name:** `{{from_name}}`

   **Message / Content:**
   ```
   {{message}}
   ```

   Or use a richer HTML template:
   ```html
   <p>Dear {{to_name}},</p>
   <p>Congratulations! Your application to join InnoPitch & Greenovation Mavericks Club has been approved.</p>
   <p>Welcome aboard! We look forward to seeing you at our upcoming events and activities.</p>
   <p>Best regards,<br>The InnoPitch Team</p>
   ```

4. **Save** the template.

### Step 4: Verify EmailJS service configuration

1. In the EmailJS dashboard, go to **Email Services** and verify:
   - Service ID: `service_3s6k61q` exists and is connected to a valid email provider (Gmail, Outlook, etc.).
   - The service is **enabled** (toggle is on).
   - The connected email account is authenticated and not expired.

2. Go to **Account** → **API Keys** and verify:
   - public Key: `OTGn7vkLxLJnLZt_d` is correct and active.

### Step 5: Test the email sending

1. Open `admin.html` in a browser.
2. Log in with the admin password.
3. Find a pending registration and click **Approve**.
4. Check:
   - The toast should say "Student approved and greeting email sent successfully."
   - The recipient should receive the greeting email.
   - If it fails, open the browser DevTools Console (F12) to see the detailed error.

### Step 6: Common troubleshooting

| Symptom | Likely Cause | Fix |
|---|---|---|
| "Approval completed, but the greeting email could not be sent." | Template params don't match template variables | Use `to_email`, `to_name`, `from_name`, `subject`, `message` |
| Error in console: `Invalid service ID` | Service ID is wrong or service not configured | Verify `service_3s6k61q` in EmailJS dashboard |
| Error in console: `Invalid template ID` | Template ID is wrong | Verify `template_z7dg3ls` in EmailJS dashboard |
| Error: `Public key is invalid` | Public key is wrong or not initialized | Verify `ZvZxJ7TyTvwUWRTrp` in EmailJS dashboard |
| Email not received (no error) | Template "To" field doesn't use `{{to_email}}` | Set "To" field to `{{to_email}}` in template editor |
| Email goes to spam | No verified sender domain | Verify domain in EmailJS or use a verified Gmail account |

### Step 7: Verify the complete approval + email flow

The full flow when an admin clicks **Approve**:

1. `postAdminAction('update', id, { Status: 'Approved' })` — updates the registration status in `localStorage` under key `innopitch_registrations`.
2. `sendGreetingEmail(row.Name, row.Email)` — sends a greeting email via EmailJS to the student's email address.
3. `fetchRows()` — refreshes the dashboard table and summary counts.
4. The `members.html` page (when refreshed) will show the approved student as a member card (it filters for `Status === 'approved'`).

### Step 8: Optional — Add email status tracking

To track whether the greeting email was sent, you can add an `Email Sent` field to the registration row:

```javascript
// In the approve handler, after successful email send:
await postAdminAction('update', id, { Status: 'Approved', 'Email Sent': 'Yes' });
```

And display it in the admin table:

```javascript
// In renderTable(), add a column:
<td>${row['Email Sent'] || 'No'}</td>
```
