# OpenShift Templates & Helm Charts — Notes + Interview Q&A

> Simple English explanation with examples. Based on "Deploy Packaged Applications" (OpenShift Templates + Helm Charts).

---

## PART 1: OpenShift Templates

### 🔑 What is a Template? (Key Idea)
A **Template** is like a **form/blueprint** in Kubernetes. It describes a group of resources (Deployment, Service, Secret, Route, etc.) that you want to create together.

- You fill in **parameters** (like blanks in a form) → Template creates real resources.
- Example: One template can create a whole database (Secret + Service + StatefulSet) just by giving a few values like username/password.

> 💡 **Think of it like a Word template**: The structure is fixed, but you fill in the name, date, etc. each time.

### ⭐ Important Highlights

| Point | Why it matters |
|---|---|
| Templates live in the `openshift` namespace by default | Populated automatically by **Cluster Samples Operator** |
| `oc get templates -n openshift` | Lists all available templates |
| `oc describe template <name> -n openshift` | Shows parameters, objects, labels |
| `oc process --parameters <name>` | Shows ONLY the parameter list (quick view) |
| `oc process` | Converts template + parameter values → real YAML manifests |
| `oc new-app --template=<name>` | Quick way to deploy (good for dev/test, NOT production) |
| `oc apply -f -` | Used with `oc process` output to create/update live resources |
| Parameters can be **generated automatically** (e.g., random password using regex `[a-zA-Z0-9]{16}`) | Saves you from typing secrets manually |

### 🧭 Typical Workflow (Remember this order!)
1. **Discover** → `oc get templates -n openshift`
2. **Inspect** → `oc describe template` or `oc process --parameters`
3. **Copy/Customize** (for production) → `oc get template ... -o yaml > my-template.yaml`
4. **Upload to your project** → `oc create -f my-template.yaml`
5. **Process + Deploy** → `oc process ... | oc apply -f -`
6. **Update later** → change parameters → `oc process ... | oc diff -f -` (preview) → `oc apply -f -`

### ⚠️ Important Rule
> `oc new-app --template=` can only **CREATE** new resources — it **cannot update** existing ones.
> For updates, always use: `oc process ... | oc apply -f -`

### 🛠️ Example Commands
```bash
# See a template's parameters
oc process --parameters cache-service -n openshift

# Generate manifest file with a parameter
oc process my-cache-service -p APPLICATION_USER=user1 -o yaml > manifest.yaml

# Use a parameter file (better for production / version control)
oc process my-cache-service -o yaml --param-file=my-params.env > manifest.yaml

# Deploy directly (pipe to apply)
oc process my-cache-service --param-file=my-params.env | oc apply -f -

# Preview changes before applying (very useful!)
oc process my-cache-service -o yaml --param-file=new-params.env | oc diff -f -
```

---

## PART 2: Helm Charts

### 🔑 What is Helm? (Key Idea)
**Helm** is a tool (like a "package manager for Kubernetes" — similar to `apt` or `npm`) that helps install, upgrade, and manage applications on Kubernetes using **Charts**.

> 💡 **Think of it like installing an app from an app store**: You pick the app (chart), maybe change some settings (values), and install it (release).

### ⭐ Important Highlights — Key Terms

| Term | Simple Meaning |
|---|---|
| **Chart** | The packaged application (like an installer file) |
| **Release** | One installed copy/instance of a chart. You can install the SAME chart multiple times = multiple releases |
| **Values** | Settings you can customize (like image name, replica count, memory limits) |
| **Repository** | A place (URL) where charts are stored/shared |

### 📁 Chart Folder Structure
```
sample/
├── Chart.yaml       → metadata (name, version)
├── templates/        → actual resource definitions (Deployment, Service, etc.)
│   └── example.yaml
└── values.yaml       → default settings/values
```

### 🛠️ Important Helm Commands

| Command | What it does |
|---|---|
| `helm repo add <name> <url>` | Add a chart repository |
| `helm search repo --versions` | List all charts + versions available |
| `helm show chart <chart>` | Show chart info (name, version, maintainer) |
| `helm show values <chart>` | Show default/customizable values |
| `helm install <release-name> <chart> -f values.yaml` | Install (deploy) the chart |
| `helm install ... --dry-run` | **Preview only** — see what would be created, without actually creating it |
| `helm list` | Show installed releases |
| `helm upgrade <release-name> <chart> -f values.yaml` | Update an existing release |
| `helm history <release-name>` | See past revisions |
| `helm rollback <release-name> <revision>` | Go back to a previous version |

### ⚠️ Important Rules to Remember
- `helm upgrade` by default installs the **latest chart version** if you don't specify `--version`.
- Never manually edit/delete Helm's release **Secret** (`helm.sh/release.v1`) — Helm needs it to track releases.
- Same chart → different `values.yaml` → different environments (dev vs prod) — this is a common real-world pattern.
- Rollback isn't always 100% safe — some upgrades aren't reversible. Test rollback in a test environment first.

### 🧭 Typical Workflow
1. `helm repo add` → add the source of charts
2. `helm search repo --versions` → find the chart + version
3. `helm show values <chart>` → see what you can customize
4. Create your own `values.yaml` with only the settings you want to override
5. `helm install <release> <chart> -f values.yaml --version X` (use `--dry-run` first to preview)
6. Later: `helm upgrade` to change settings or update version
7. `helm rollback` if something breaks

### 🛠️ Example: Overriding Only What You Need
Default `values.yaml` from chart:
```yaml
image:
  repository: "sample"
  tag: "1.8.10"
  pullPolicy: IfNotPresent
```
Your custom file (only override the tag):
```yaml
image:
  tag: "1.8.10-patched"
```

---

## 🆚 Templates vs Helm Charts — Quick Comparison

| Feature | OpenShift Template | Helm Chart |
|---|---|---|
| Native to | OpenShift only | Kubernetes (works everywhere) |
| Format | Single YAML file with parameters | Folder with multiple files (Chart.yaml, templates/, values.yaml) |
| Logic/templating | Simple parameter substitution | Full Go templating (loops, conditions, functions) |
| Update tracking | Manual (`oc diff`, `oc apply`) | Built-in (`helm upgrade`, `helm history`, `helm rollback`) |
| Best for | Simple app packaging | Complex apps, versioned releases, hooks |
| Command tool | `oc` | `helm` |

---

## 📋 Interview Questions & Answers (Easy English)

### Q1: What is an OpenShift Template?
**A:** A Template is like a **blueprint** that describes a group of Kubernetes resources (like Deployment, Service, Secret). It has **parameters** — blank spaces you fill in — so you can reuse the same template to create different apps with different names, passwords, or images.

*Example:* A `mysql-persistent` template can create a MySQL database. You just give it a username and password, and it builds the Secret, Service, and StatefulSet for you.

---

### Q2: How do you see what parameters a template needs?
**A:** Use:
```bash
oc process --parameters <template-name> -n openshift
```
This shows just the list of parameters (name, description, default value) — faster than reading the whole template.

---

### Q3: What is the difference between `oc new-app --template=` and `oc process | oc apply`?
**A:** 
- `oc new-app --template=` → Quick and easy, but can **only create new resources**. Good for testing.
- `oc process | oc apply` → Converts the template into a manifest and applies it. This can **also update existing resources**, so it's the right choice for production.

*Example:* If you already deployed an app and want to change its memory limit, `oc new-app` won't help — you must use `oc process ... | oc apply -f -`.

---

### Q4: How can you preview changes before applying them to a live template-based app?
**A:** Use `oc diff`:
```bash
oc process my-app -o yaml --param-file=new-params.env | oc diff -f -
```
It shows a "before vs after" comparison (like `git diff`), so you can check exactly what will change before you apply it.

---

### Q5: What is Helm?
**A:** Helm is a **package manager for Kubernetes**. It helps you install, upgrade, and manage applications easily using packages called **Charts**. It's similar to how `apt-get` installs software on Linux, but for Kubernetes apps.

---

### Q6: What is the difference between a Helm "Chart" and a "Release"?
**A:** 
- **Chart** = the package/template of the application (not yet running).
- **Release** = one actual deployed instance of that chart.

*Example:* You can install the `etherpad` chart twice — once as release `test` and once as release `prod` — both running independently from the same chart.

---

### Q7: How do you customize a Helm chart deployment without editing the chart itself?
**A:** By creating your own `values.yaml` file with only the settings you want to change, then passing it during install/upgrade:
```bash
helm install myapp mychart -f values.yaml
```
You don't need to copy the whole chart — just override specific keys.

---

### Q8: What does `helm upgrade` do, and what should you be careful about?
**A:** `helm upgrade` updates an existing release with new values or a new chart version. 
⚠️ Careful point: By default, if you don't specify `--version`, it uses the **latest available chart version**, which might introduce unexpected changes. Always check chart release notes and use `--dry-run` to preview first.

---

### Q9: How do you roll back a Helm release if an upgrade breaks something?
**A:** 
```bash
helm history myapp        # see all past revisions
helm rollback myapp 1     # go back to revision 1
```
This restores the app to a previous known-good state. Note: not all changes are reversible, so test rollback in a non-production environment first.

---

### Q10: How does OpenShift populate its default templates?
**A:** The **Cluster Samples Operator** automatically adds sample templates (and image streams) into the `openshift` namespace when the cluster is set up. Admins can opt-out or restrict which templates get added.

---

### Q11: Why would you copy a template into your own project instead of using it directly from `openshift` namespace?
**A:** For production use, you want a **stable, customized copy** — with your own default values, name, and without unwanted namespace references — so it's tracked properly and doesn't change if the shared template is modified later.

*Steps:*
```bash
oc get template cache-service -o yaml -n openshift > my-cache-service.yaml
# edit the file: rename, adjust defaults, remove namespace field
oc create -f my-cache-service.yaml
```

---

### Q12: What is inside a Helm chart folder?
**A:**
- `Chart.yaml` → basic info (name, version)
- `templates/` → the actual Kubernetes resource definitions (using Go templating)
- `values.yaml` → default configuration values

---

### Q13: How can you add resource limits (CPU/memory) using Helm without changing the chart source code?
**A:** Add it in your custom `values.yaml`:
```yaml
resources:
  limits:
    memory: 256Mi
  requests:
    memory: 128Mi
```
Then run `helm upgrade myapp mychart -f values.yaml`. The chart already supports this key — you're just filling it in.

---

### Q14: What happens if you delete a Helm release's Secret manually?
**A:** You should **never do this**. Helm stores release state/history as a Secret (`helm.sh/release.v1` type). If deleted, Helm loses track of that release and can no longer manage (upgrade/rollback) it properly.

---

### Q15: When would you use OpenShift Templates vs Helm Charts?
**A:** 
- Use **Templates** for simple, OpenShift-only apps with basic parameter substitution.
- Use **Helm Charts** for more complex applications that need versioning, rollback, hooks, and cross-platform (any Kubernetes) support — Helm is the more powerful and widely-used industry standard.

---

## 📝 Quick Revision Summary (One-liners)

- Template = OpenShift blueprint + parameters → `oc process` + `oc apply`
- Helm Chart = packaged app → `helm install` / `helm upgrade` / `helm rollback`
- Always **preview before applying**: `oc diff` (templates) or `--dry-run` (Helm)
- Production tip: use parameter files / values files + version control, not command-line flags
- Never touch Helm's internal release Secret
