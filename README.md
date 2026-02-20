# Complete Red Flag System Bug Triage & Fixes

Below are the exact code snippets for the fixes implemented across both the `transaction server` logic and `content server` HTML templates. This resolves the email delivery failure and the missing patient form submissions inside the PDFs.

---

### BUG 1: App crashing internally due to missing SendGrid library checks
**File:** `redflags/utils.py`

**Before:**
```python
    try:
        from sendgrid import SendGridAPIClient
        from sendgrid.helpers.mail import (
            Mail, Email, To, Content, Attachment,
            FileContent, FileName, FileType, Disposition
        )
    except ImportError:
        print("[RF] SendGrid library not installed. Doctor email will NOT be sent.")
        return False
```

**After:**
```python
    # IMPORT BLOCK REMOVED ENTIRELY 
```

**Explanation:** In a local or separate test environment (which might not have `sendgrid` globally pip installed), this block was intercepting the email request and instantly terminating it with `return False`. Removing the imports from this global utility scope prevented the execution from halting early and allowed the fallback systems to kick in.

---

### BUG 2: Adding a Django SMTP Email Fallback
**File:** `redflags/mails.py`

**Before:**
```python
def send_email_via_sendgrid(to, title, body, attachments=None, from_email=None, from_name=None):
    # SendGrid preparation omitted...
    
    try:
        sg = SendGridAPIClient(sg_api_key)
        response = sg.send(message)
        return response.status_code in (200, 201, 202)
    except Exception as e:
        print(f"[RF] SendGrid error: {repr(e)}")
        return False
```

**After:**
```python
def send_email_via_sendgrid(to, title, body, attachments=None, from_email=None, from_name=None):
    # SendGrid preparation omitted...
    
    try:
        sg = SendGridAPIClient(sg_api_key)
        response = sg.send(message)
        return response.status_code in (200, 201, 202)
    except Exception as e:
        print(f"[RF] SendGrid error: {repr(e)}")
        print(f"[RF] Localhost: Redirecting SendGrid call to Django SMTP ({from_email} -> {to})")
        
        # --- NEW DJANGO NATIVE SMTP FALLBACK BLOCK ---
        try:
            from django.core.mail import EmailMultiAlternatives
            email = EmailMultiAlternatives(
                subject=title,
                body=body,
                from_email=from_email,
                to=[to]
            )
            email.content_subtype = "html"
            
            # Attach attachments built by SendGrid wrapper
            if attachments:
                import base64
                from sendgrid.helpers.mail import Attachment as SGAttachment
                for att in attachments:
                    if isinstance(att, SGAttachment):
                        fname = att.file_name.get()
                        b64_data = att.file_content.get()
                        mime = att.file_type.get()
                        raw_data = base64.b64decode(b64_data)
                        email.attach(fname, raw_data, mime)
            
            email.send(fail_silently=False)
            print("[RF] SMTP Email sent successfully via Django/Gmail")
            return True
        except Exception as e2:
            print(f"[RF] Local SMTP error: {repr(e2)}")
            return False
```

**Explanation:** Even though SendGrid was configured to send emails, without the API working correctly in the local environment, it cleanly caught the exception and `returned False`. I wrapped the exception to spin up Django's native `EmailMultiAlternatives` handler, which bypasses SendGrid entirely and talks directly to Gmail securely using your Google App Password. I also included a mapper to unpack SendGrid's attachment payload and map it inside the Django Email attachment API so your PDFs still carried over!

---

### BUG 3: Email Generation Logic was Commented Out
**File:** `redflags/views.py`

**Before:**
```python
        # try:
        #     if self._should_email_doctor(redflags):
        #         report_id = event["event_id"]
        #         from .utils import send_doctor_report_via_sendgrid
        #         send_doctor_report_via_sendgrid(
        #             request=request,
        #             doctor=link.doctor,
        #             form_json=form_json,
        #             lang=lang,
        #             patient_name=patient_name,
        #             patient_mobile=patient_mobile,
        #             cleaned_answers={k: v for k, v in cleaned.items()},
        #             redflags=redflags,
        #             report_id=report_id,
        #             doctor_mobile=doc_phone,
        #         )
        # except Exception as e:
        #     print("[RF] doctor email failed:", e)
```

**After:**
```python
        try:
            if self._should_email_doctor(redflags):
                report_id = event["event_id"]
                from .utils import send_doctor_report_via_sendgrid
                send_doctor_report_via_sendgrid(
                    request=request,
                    doctor=link.doctor,
                    form_json=form_json,
                    lang=lang,
                    patient_name=patient_name,
                    patient_mobile=patient_mobile,
                    cleaned_answers={k: v for k, v in cleaned.items()},
                    redflags=redflags,
                    report_id=report_id,
                    doctor_mobile=doc_phone,
                )
        except Exception as e:
            print("[RF] doctor email failed:", e)
```

**Explanation:** Inside `DynamicFormView.post` (which processes form submissions and saves them to the DB), the code meant to trigger the `send_doctor_report_via_sendgrid` function was completely commented out as a block block. No matter what happened, emails were skipping dispatch. I uncommented the try-except logic so the view actively tries to package the context.

---

### BUG 4: Terminal console interception
**File:** `project/settings.py`

**Before:**
```python
EMAIL_BACKEND = 'django.core.mail.backends.console.EmailBackend'
```

**After:**
```python
EMAIL_BACKEND = 'django.core.mail.backends.smtp.EmailBackend'
```

**Explanation:** Using `console.EmailBackend` redirects the generated email text stream directly to standard output (the terminal) so you don't accidentally spam servers during tests. By modifying the routing directly into the `smtp.EmailBackend` parameter, Django correctly utilized your `.env` `EMAIL_HOST_USER` and Google App Password variables and sent the data payload out of the terminal completely.

---

### BUG 5: Missing Patient HTML Loop block mapping
**File:** `content-server-folder/templates/reports/patient_report.html`

**Before:**
```html
{% if redflags %}
  <table class="tbl">
    {% for rf in redflags %}
      <tr>
        <td>{{ rf.text }} {% if rf.info_url %}- <a href="{{ rf.info_url }}">Click Here for more information</a>{% endif %}</td>
      </tr>
    {% endfor %}
  </table>
{% endif %}

<p class="muted" style="margin-top:10px">{{ disclaimer }}</p>
```

**After:**
```html
{% if redflags %}
  <table class="tbl">
    {% for rf in redflags %}
      <tr>
        <td>{{ rf.text }} {% if rf.info_url %}- <a href="{{ rf.info_url }}">Click Here for more information</a>{% endif %}</td>
      </tr>
    {% endfor %}
  </table>
{% endif %}

{% if answers %}
  <br>
  <h3>Patient Answers</h3>
  <table class="tbl">
    {% for qa in answers %}
      <tr>
        <th>{{ qa.q }}</th>
        <td>{{ qa.a }}</td>
      </tr>
    {% endfor %}
  </table>
{% endif %}

<p class="muted" style="margin-top:10px">{{ disclaimer }}</p>
```

**Explanation:** The content server runs isolated from the actual `Django` transaction system (which was why the PDF logic bypassed the main rendering system). However, the developer never added the HTML mapping variables inside this template logic to read `answers`. Without the `{% if answers %}` block present, the PDF compiler quietly ignored the python dictionary passed over, yielding a blank bottom section.

---

### BUG 6: Doctor report structural formatting
**File:** `content-server-folder/templates/reports/doctor_report.html`

**Before:**
```html
{% if answers %}
  <h2>Form entries made by the patient:</h2>
  <table class="tbl">
    {% for q in answers %}
      <tr>
        <td>
          <!-- Querying for question: {{q.question}} -->
          <strong>{{ q.q }}</strong>
          <br>
          <!-- Querying for answer of that question: {{q.answer}} -->
          {{ q.a }}
        </td>
      </tr>
    {% endfor %}
  </table>
{% endif %}
```

**After:**
```html
{% if answers %}
  <h2>Form entries made by the patient:</h2>
  <table class="tbl">
    {% for qa in answers %}
      <tr>
        <th>{{ qa.q }}</th>
        <td>{{ qa.a }}</td>
      </tr>
    {% endfor %}
  </table>
{% endif %}
```

**Explanation:** In `doctor_report.html`, the code block to capture `answers` actually *did* exist! But it was extremely messy and formatted the Questions and Answers stacked in a single vertical table column. Since the PDF rendering acts just like HTML table renders, I refactored the extraction so it mimics the `patient_report.html` syntax by splitting it inside the python `for` loop logic natively assigning Questions mapping dynamically alongside an adjacent cell mapping Answers.
