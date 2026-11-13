# OpenShift Templates and Helm Charts: Notes and Interview Q&A

This chapter explains two ways to package and deploy applications: OpenShift Templates and Helm Charts. The examples use simple language and common commands.

---

## Part 1: OpenShift Templates

### What is an OpenShift Template?

An OpenShift **Template** is a reusable description of one or more resources. A resource is an object that runs or supports an application, such as a Deployment, Service, Secret, or Route.

A template can contain **parameters**. Parameters are values you provide when you use the template, such as an application name or database username. OpenShift fills in those values to produce resource definitions.

For example, one database template might describe a Secret, Service, and StatefulSet. You can provide different names or credentials each time you process it.

> Think of a template as a form: the layout is prepared, and you fill in its blank fields.

### Important points

| Point | What it means |
|---|---|
| Sample templates may be in the `openshift` namespace | The Cluster Samples Operator can install sample templates there. Which samples are available depends on cluster configuration. |
| `oc get templates -n openshift` | Lists templates in the `openshift` namespace. |
| `oc describe template <name> -n openshift` | Shows information about a template, including its parameters and objects. |
| `oc process --parameters <name>` | Lists a template's parameters. Add `-n <namespace>` if needed. |
| `oc process` | Fills in template parameters and prints the resulting resource definitions. |
| `oc new-app --template=<name>` | A shortcut for creating an application from a template. It is often useful for development and testing. |
| `oc apply -f -` | Reads resource definitions from standard input and creates or updates resources to match them. |
| Generated parameters | A template can generate values, such as a random password, when configured to do so. |

### A typical workflow

1. **Find a template:** `oc get templates -n openshift`
2. **Inspect it:** `oc describe template <name> -n openshift` or `oc process --parameters <name> -n openshift`
3. **Customize it if needed:** `oc get template <name> -n openshift -o yaml > my-template.yaml`
4. **Add it to your project if needed:** `oc create -f my-template.yaml`
5. **Process and apply it:** `oc process ... | oc apply -f -`
6. **Preview later changes:** process the template and pipe the result to `oc diff -f -` before applying it.

### Creating resources and updating them

`oc new-app --template=<name>` is a convenient way to create an application from a template. It is not a release-management or reconciliation command, so do not rely on it to update resources from a previous deployment. If a resource already exists, the result depends on the command and the resource; name conflicts can prevent creation.

For repeatable updates, process the template into resource definitions and use `oc apply`. Review the changes first with `oc diff` when appropriate. `oc apply` can create resources that are missing and update existing ones.

### Example commands

```bash
# See a template's parameters
oc process --parameters cache-service -n openshift

# Generate a manifest file with a parameter
oc process my-cache-service -p APPLICATION_USER=user1 -o yaml > manifest.yaml

# Use a parameter file
oc process my-cache-service -o yaml --param-file=my-params.env > manifest.yaml

# Process a template and apply its resources
oc process my-cache-service --param-file=my-params.env | oc apply -f -

# Preview changes before applying them
oc process my-cache-service -o yaml --param-file=new-params.env | oc diff -f -
```

---

## Part 2: Helm Charts

### What is Helm?

**Helm** is a tool for installing and managing applications on Kubernetes. It uses packages called **Charts**. Helm can install a chart, change an installation, and help you roll back to an earlier revision.

> Think of a chart as an application package. You choose settings and install it as a release.

### Important terms

| Term | Simple meaning |
|---|---|
| **Chart** | A package of templates and default settings for an application. |
| **Release** | An installed instance of a chart. You can install the same chart more than once using different release names. |
| **Values** | Settings used to configure a chart, such as the image name, replica count, or resource limits. |
| **Repository** | A server or location where charts are stored and shared. |

### Example chart folder

```text
sample/
├── Chart.yaml        # Chart information, including its name and version
├── templates/        # Resource definitions, often written with Go templates
│   └── example.yaml
└── values.yaml       # Default settings
```

### Useful Helm commands

| Command | What it does |
|---|---|
| `helm repo add <name> <url>` | Adds a chart repository. |
| `helm search repo --versions` | Lists charts and their available versions in configured repositories. |
| `helm show chart <chart>` | Shows chart information, such as its name and version. |
| `helm show values <chart>` | Shows the chart's default settings. |
| `helm install <release-name> <chart> -f values.yaml` | Installs a chart as a release, using settings from a file. |
| `helm install ... --dry-run` | Shows what Helm would do without installing the release. |
| `helm list` | Lists installed releases. |
| `helm upgrade <release-name> <chart> -f values.yaml` | Updates an existing release using the chart and settings provided. |
| `helm history <release-name>` | Shows earlier revisions of a release. |
| `helm rollback <release-name> <revision>` | Tries to return a release to an earlier revision. |

### Important points

- `helm upgrade <release> <chart>` uses the chart specified in that command. If `<chart>` is a chart reference from a repository and you do not give `--version`, Helm selects the latest available chart version from that repository. Use `--version <chart-version>` to select a specific repository chart version.
- A local chart directory or package is the chart you provide directly; `--version` is not a way to choose a different repository version for it.
- Do not manually edit or delete Helm's release records. Helm 3 stores these as Secrets by default, but a different storage driver can be configured.
- The same chart can be installed with different values for different environments, such as development and production.
- A rollback may not undo every effect of an upgrade. For example, external changes or data changes may remain. Test rollback before relying on it in production.

### A typical workflow

1. Add a repository with `helm repo add`.
2. Find a chart and its versions with `helm search repo --versions`.
3. Review its settings with `helm show values <chart>`.
4. Create a `values.yaml` file containing the settings you want to override.
5. Preview and install the chart. Use `--version` to choose a particular repository chart version.
6. Later, run `helm upgrade <release> <chart>` with the chart you want to use and the values you want.
7. If needed, use `helm rollback` to try to return to an earlier release revision.

### Example: Override only one setting

Suppose the chart's default `values.yaml` contains:

```yaml
image:
  repository: "sample"
  tag: "1.8.10"
  pullPolicy: IfNotPresent
```

To change only the image tag, your own values file can contain:

```yaml
image:
  tag: "1.8.10-patched"
```

The chart must support the setting you want to change.

---

## OpenShift Templates and Helm Charts: Comparison

| Feature | OpenShift Template | Helm Chart |
|---|---|---|
| Common use | OpenShift environments | Kubernetes environments, including OpenShift |
| Format | A Template resource containing parameters and resource definitions | A chart folder or package with chart metadata, templates, and default values |
| Customization | Parameters | Values |
| Template logic | Parameter substitution | Go templating, including conditions, loops, and functions |
| Updates | Commonly process again, inspect with `oc diff`, and apply with `oc apply` | Manage a named release with `helm upgrade`, history, and rollback commands |
| Often a good fit for | Simple applications designed for OpenShift | Applications that need chart packaging and release management |
| Main command-line tool | `oc` | `helm` |

OpenShift Templates are an OpenShift feature and are not as portable as Helm charts across different Kubernetes platforms. The best choice depends on the application and the tools your team uses.

---

## Interview Questions and Answers

### Q1: What is an OpenShift Template?

**A:** It is a reusable description of one or more resources, such as a Deployment, Service, or Secret. It can have parameters that you fill in to create resource definitions for a particular application.

*Example:* A database template might define a Secret, Service, and StatefulSet. You provide values such as a username and database name.

### Q2: How do you see which parameters a template needs?

**A:** Run:

```bash
oc process --parameters <template-name> -n <namespace>
```

This lists the parameter names and, when defined, their descriptions and default values.

### Q3: What is the difference between `oc new-app --template` and `oc process | oc apply`?

**A:** `oc new-app --template=<name>` is a shortcut for creating an application from a template. It is not a reliable way to update resources from an earlier deployment. For repeatable updates, use `oc process` to produce the resource definitions and `oc apply` to create or update them.

```bash
oc process <template-name> -o yaml | oc apply -f -
```

Check for differences first with `oc diff` if you want to review the update.

### Q4: How can you preview changes to a template-based application?

**A:** Process the template and send the output to `oc diff`:

```bash
oc process my-app -o yaml --param-file=new-params.env | oc diff -f -
```

It compares the resource definitions with the resources in the cluster. Review the output before applying changes.

### Q5: What is Helm?

**A:** Helm is a tool for packaging and managing applications on Kubernetes. It uses charts to install applications and manage their releases.

### Q6: What is the difference between a chart and a release?

**A:** A **chart** is the application package. A **release** is an installed instance of that chart.

*Example:* You can install one chart as `test` and again as `prod`, with different settings for each release.

### Q7: How do you customize a chart without editing it?

**A:** Put the settings you want to change in your own values file and pass it to Helm:

```bash
helm install myapp mychart -f values.yaml
```

The chart must support the values you provide.

### Q8: What does `helm upgrade` do?

**A:** It updates an existing release using the chart and settings named in the command. For example:

```bash
helm upgrade myapp mychart -f values.yaml
```

For a chart reference from a repository, add `--version <chart-version>` when you want a specific chart version. Without it, Helm uses the latest available version from that repository. The command does not automatically choose a new chart if you supplied a local chart path.

### Q9: How can you roll back a Helm release?

**A:** View the release history and choose a revision:

```bash
helm history myapp
helm rollback myapp 1
```

Rollback tries to restore an earlier release revision. It may not undo changes outside Helm or changes to application data, so test it safely first.

### Q10: How are sample templates added to OpenShift?

**A:** The Cluster Samples Operator can install sample templates and image streams in the `openshift` namespace. The available samples depend on the cluster's configuration.

### Q11: Why copy a template into your project?

**A:** A project-owned copy lets you customize and manage the template yourself. It can also reduce reliance on changes to a shared sample. Review the copied YAML, remove or change any `metadata.namespace: openshift` field so it belongs to your project, and adjust it for your cluster before creating it.

```bash
oc get template cache-service -o yaml -n openshift > my-cache-service.yaml
# Remove/change metadata.namespace if it is set to openshift, then create the template
oc create -f my-cache-service.yaml
```

### Q12: What is inside a Helm chart?

**A:**

- `Chart.yaml` contains chart information, such as its name and version.
- `templates/` contains resource definitions. These can use Go templates.
- `values.yaml` contains default configuration values.

### Q13: How can you set CPU or memory limits with Helm?

**A:** Add the settings to your custom values file, if the chart supports them:

```yaml
resources:
  limits:
    memory: 256Mi
  requests:
    memory: 128Mi
```

Then pass the file to `helm upgrade`:

```bash
helm upgrade myapp mychart -f values.yaml
```

### Q14: What happens if you delete a Helm release Secret manually?

**A:** Helm 3 uses Secrets by default to store release information. Deleting a release record can make Helm lose the information it needs to manage that release. Do not edit or delete these records manually unless you understand the consequences and have a recovery plan.

### Q15: When would you use an OpenShift Template or a Helm chart?

**A:** An OpenShift Template can suit a simple application that is designed for OpenShift and needs basic parameter substitution. A Helm chart can suit an application that needs chart packaging, named releases, and Helm's upgrade and history commands. Consider portability and your team's existing tools when choosing.

---

## Quick Revision Notes

- **Template:** OpenShift resource definitions with parameters. Use `oc process` to fill them in.
- **Template deployment:** Use `oc apply` to create or update resources; use `oc diff` to preview differences.
- **Helm chart:** An application package with templates and default values.
- **Helm release:** An installed instance of a chart.
- **Helm commands:** `helm install`, `helm upgrade`, `helm history`, and `helm rollback`.
- **Chart version:** `helm upgrade` uses the chart specified in the command. For a repository chart, use `--version` to choose a specific version.
- Keep parameter and values files under suitable version control, and handle secrets carefully.
- Do not manually change Helm's release records.
