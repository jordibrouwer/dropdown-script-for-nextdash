# dropdown-script-for-nextdash
# NextDash Dropzone 5 Action

> Seamlessly save URLs from macOS to your self-hosted [NextDash](https://github.com/jordibrouwer/nextdash) instance with a single click or drag-and-drop gesture using **Dropzone 5**.

![NextDash Dropzone Integration](https://nextdash.cc/wp-content/uploads/2026/08/dropzone-5-is-here-v0-64nlcmfvysqg1.png-300x300.webp)

As the creator of [NextDash](https://github.com/jordibrouwer/nextdash), I built this custom **Dropzone 5 Action** for macOS to eliminate friction when organizing the web. NextDash is a fast, clean, self-hosted dashboard for managing bookmarks, services, and personal inbox items. 

With this Dropzone action, you can capture any URL from Safari, Chrome, or Firefox and push it directly into your NextDash inbox with a single click or a quick drag-and-drop gesture—without interrupting your browsing workflow.

---

## Why Use a Dropzone Action for NextDash?

Dropzone 5 is a productivity utility for macOS that sits in your menu bar and speeds up file sharing, application launching, and macro execution. Pairing it with NextDash gives you a native, desktop-level shortcut to capture web content.

- **Zero-Context Switching:** Save links without opening your dashboard tab or leaving your current browser session.
- **Two Ways to Capture:** Drag a URL directly onto the Dropzone icon, or copy a URL to your clipboard and click the Dropzone action tile.
- **Inbox-First Workflow:** Keeps your reading workflow organized by routing incoming links directly to NextDash's `/api/inbox` endpoint so you can classify or move them later.

---

## How It Works Under the Hood

NextDash provides a lean REST API for inserting items into your dashboard. This Dropzone action uses a native **Ruby script** (`action.rb`) bundled inside a `.dzbundle` container to interact with that API:

1. **Input Detection:** When triggered via click, the script reads your macOS clipboard using Dropzone's `$dz.read_clipboard` API. When triggered via drag-and-drop, it grabs the dragged text or link from `$items[0]`.
2. **Payload Construction:** The script formats the URL into a lightweight JSON payload: `{"url": "https://example.com"}`.
3. **Authentication & HTTP POST:** It executes an HTTPS POST request to your self-hosted NextDash endpoint (`/api/inbox`). If you have configured a `NEXTDASH_WRITE_TOKEN` in your NextDash environment variables, the script attaches it seamlessly in the `X-NextDash-Token` HTTP request header.
4. **Visual Feedback:** Dropzone provides real-time notifications in the menu bar (`$dz.finish` or `$dz.fail`), letting you know instantly whether the bookmark was saved successfully.

---

## Quick Installation (Automated)

If you are running Dropzone 5 on macOS, you can set up the action in seconds using a single Terminal command.

Open your Terminal app, paste the following command, and hit **Enter**:

```bash
python3 -c "
import os

p1 = os.path.expanduser('~/Library/Application Support/Dropzone/Actions/NextDash.dzbundle')
p2 = os.path.expanduser('~/Library/Application Support/Dropzone 5/Actions/NextDash.dzbundle')

code = '''# Dropzone Action Info
# Name: NextDash Bookmark
# Description: Saves a URL to NextDash via the API
# Handles: Text, URLs
# Creator: Jordi Brouwer
# URL: https://github.com/jordibrouwer/nextdash
# Events: Clicked, Dragged
# SkipConfig: No
# Version: 1.0
# MinDropzoneVersion: 4.0

require 'net/http'
require 'uri'
require 'json'

# Configuration
NEXTDASH_URL = "https://nextdash.example.com/api/inbox"
WRITE_TOKEN = "" # Optional: Set your NEXTDASH_WRITE_TOKEN if enabled

def save_to_nextdash(url_string)
  \$dz.begin("Saving bookmark to NextDash...")
  uri = URI.parse(NEXTDASH_URL)
  request = Net::HTTP::Post.new(uri)
  request.content_type = "application/json"
  request["X-NextDash-Token"] = WRITE_TOKEN unless WRITE_TOKEN.empty?
  request.body = JSON.dump({ url: url_string.strip })
  req_options = { use_ssl: uri.scheme == "https" }
  begin
    response = Net::HTTP.start(uri.hostname, uri.port, req_options) { |http| http.request(request) }
    if [200, 201].include?(response.code.to_i)
      \$dz.finish("Bookmark saved to NextDash!")
    else
      \$dz.fail("Error: HTTP #{response.code}")
    end
  rescue => e
    \$dz.fail("Connection error: #{e.message}")
  end
end

def dragged
  \$dz.begin("Processing URL...")
  save_to_nextdash(\$items[0])
end

def clicked
  txt = \$dz.read_clipboard
  if txt && (txt.start_with?("http://") || txt.start_with?("https://"))
    save_to_nextdash(txt)
  else
    \$dz.fail("No valid URL found on clipboard.")
  end
end
'''

for p in [p1, p2]:
    os.makedirs(p, exist_ok=True)
    with open(os.path.join(p, 'action.rb'), 'w') as f:
        f.write(code)
    py_file = os.path.join(p, 'action.py')
    if os.path.exists(py_file):
        os.remove(py_file)

print('NextDash action bundle created successfully!')
"
```

### Post-Installation Steps

1. Restart **Dropzone 5**.
2. Open the actions folder in Finder:
   ```bash
   open "~/Library/Application Support/Dropzone 5/Actions"
   ```
3. Double-click or drag `NextDash.dzbundle` into your active Dropzone grid.
4. Edit `action.rb` inside the bundle to update `NEXTDASH_URL` with your actual NextDash endpoint (e.g., your local IP, Tailscale URL, or Nginx Proxy Manager domain) and optional `WRITE_TOKEN`.

---

## Action Source Code (`action.rb`)

If you prefer to inspect or customize the Ruby code manually, create a folder named `NextDash.dzbundle` in your Dropzone actions directory (`~/Library/Application Support/Dropzone 5/Actions/`) and save the code below as `action.rb`:

```ruby
# Dropzone Action Info
# Name: NextDash Bookmark
# Description: Saves a URL to NextDash via the API
# Handles: Text, URLs
# Creator: Jordi Brouwer
# URL: https://github.com/jordibrouwer/nextdash
# Events: Clicked, Dragged
# SkipConfig: No
# Version: 1.0
# MinDropzoneVersion: 4.0

require 'net/http'
require 'uri'
require 'json'

# --- Configuration ---
# Replace with your actual NextDash instance URL (Tailscale, Nginx proxy, or local IP)
NEXTDASH_URL = "https://nextdash.example.com/api/inbox"

# Optional: Add your NEXTDASH_WRITE_TOKEN if your instance requires token auth
WRITE_TOKEN = ""

def save_to_nextdash(url_string)
  $dz.begin("Saving bookmark to NextDash...")
  
  uri = URI.parse(NEXTDASH_URL)
  request = Net::HTTP::Post.new(uri)
  request.content_type = "application/json"
  
  unless WRITE_TOKEN.empty?
    request["X-NextDash-Token"] = WRITE_TOKEN
  end
  
  request.body = JSON.dump({ url: url_string.strip })

  req_options = {
    use_ssl: uri.scheme == "https"
  }

  begin
    response = Net::HTTP.start(uri.hostname, uri.port, req_options) do |http|
      http.request(request)
    end

    if response.code.to_i == 200 || response.code.to_i == 201
      $dz.finish("Bookmark saved to NextDash!")
    else
      $dz.fail("Error: HTTP #{response.code}")
    end
  rescue => e
    $dz.fail("Connection error: #{e.message}")
  end
end

def dragged
  $dz.begin("Processing URL...")
  url = $items[0]
  save_to_nextdash(url)
end

def clicked
  clipboard_text = $dz.read_clipboard
  if clipboard_text && (clipboard_text.start_with?("http://") || clipboard_text.start_with?("https://"))
    save_to_nextdash(clipboard_text)
  else
    $dz.fail("No valid URL found on clipboard.")
  end
end
```

---

## How to Use It

Once configured, capturing links takes less than a second:

- **Method 1 (Clipboard Click):** Copy any web link (`Cmd + C`) while browsing in Safari, Chrome, or Firefox. Click the **NextDash Bookmark** icon in your Dropzone grid. The action reads your clipboard and sends the URL straight to your NextDash inbox.
- **Method 2 (Drag & Drop):** Highlight a URL or link element in your browser, drag it up to the macOS menu bar to open Dropzone, and drop it onto the **NextDash Bookmark** action tile.

---

## License & Links

- **Main Project:** [NextDash GitHub Repository](https://github.com/jordibrouwer/nextdash)
- **Blog Post:** [Seamless Bookmarking: How to Save URLs to NextDash Using Dropzone 5 on macOS](https://nextdash.cc/2026/08/09/seamless-bookmarking-how-to-save-urls-to-nextdash-using-dropzone-5-on-macos/)
- **Dropzone App:** [Aptonic Dropzone 5](https://aptonic.com)
