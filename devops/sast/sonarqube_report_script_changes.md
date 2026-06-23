# Proposed Changes to `sonarqube-report.py`

This document outlines the necessary modifications to the `sonarqube-report.py` script to properly fetch and include **Security Hotspot** data from SonarQube in the `key_data.json` report. This will address the discrepancy where PowerBI reports 100% security OK despite existing security hotspots in SonarQube.

The `key_data.json` currently only fetches `BUG`, `CODE_SMELL`, and `VULNERABILITY` issue types. Security Hotspots are a distinct issue type in SonarQube and need a separate API call.

---

## 1. Add `security_hotspot_data` Fetch Logic

Locate the section where the script makes API calls for `BUG`, `CODE_SMELL`, and `VULNERABILITY`. Add a new block for `SECURITY_HOTSPOT` issues.

**Original Section (Example Snippet, `key_data.json` part):**
```python
# Se hace una solicitud a la API de SonarQube para obtener la información de vulnerabilidades del proyecto
url = f'{sonarqube_url}/api/issues/search?facets=severities&componentKeys={project_key}&types=VULNERABILITY&issueStatuses=OPEN,CONFIRMED,ACCEPTED'
response = requests.get(url, auth=(api_token,''))

if response.status_code == 200:
        data = response.json()

        # Se crea la estructura JSON que almacenará la información de las vulnerabilidades en key_data.json
        vuln_data = {
                "BUILD": build_number,
                "Type": "VULNERABILITIES",
                "Key": project_key,
                "Name": project_name,
                "Fecha": current_date,
                "facets": data["facets"]
        }

else:
        print(f'Error al hacer la solicitud: {response.status_code}')
```

**Proposed Change: Add New API Call for `SECURITY_HOTSPOT`**

Insert the following code block directly after the `VULNERABILITY` API call section:

```python
# --- ADD THIS NEW SECTION ---

# Se hace una solicitud a la API de SonarQube para obtener la información de Security Hotspots del proyecto
url = f'{sonarqube_url}/api/issues/search?facets=severities&componentKeys={project_key}&types=SECURITY_HOTSPOT&issueStatuses=OPEN,CONFIRMED,ACCEPTED'
response = requests.get(url, auth=(api_token,''))

# Initialize security_hotspot_data to avoid 'not defined' error if API call fails
security_hotspot_data = None 

if response.status_code == 200:
        data = response.json()

        # Se crea la estructura JSON que almacenará la información de los Security Hotspots en key_data.json
        security_hotspot_data = {
                "BUILD": build_number,
                "Type": "SECURITY_HOTSPOT",
                "Key": project_key,
                "Name": project_name,
                "Fecha": current_date,
                "facets": data["facets"]
        }

else:
        print(f'Error al hacer la solicitud de Security Hotspot: {response.status_code}')

# --- END NEW SECTION ---
```

---

## 2. Include `security_hotspot_data` in `key_data.json` List

Locate the section where `key_data.json` is created or updated. You need to append the newly fetched `security_hotspot_data` to the list that is dumped into the JSON file.

**Original Section (`key_data.json` part):**
```python
# ... (rest of the script)

# Si el archivo key_data.json ya existe en el path correspondiente se agregan los datos nuevos en caso contrario se crea el archivo con los datos obtenidos
    if os.path.exists(key_data_file_path):
            with open(key_data_file_path, 'r') as file:
                    existing_data = json.load(file)
                    existing_data.append(bug_data)
                    existing_data.append(codesmell_data)
                    existing_data.append(vuln_data) # <-- Add hotspot data here

            with open(key_data_file_path, 'w') as file:
                    json.dump(existing_data, file, indent=4)
                    print('El archivo key_data.json fue actualizado exitosamente.')

    else:
            with open(key_data_file_path,'w') as file:
                    key_data_list = [bug_data, codesmell_data, vuln_data] # <-- Add hotspot data here
                    json.dump(key_data_list, file, indent=4)
                    print('El archivo key_data.json fue creado exitosamente.')

# ... (rest of the script)
```

**Proposed Change: Append `security_hotspot_data`**

Modify the two `key_data_list` assignments to include `security_hotspot_data` (after checking if it exists):

```python
# ... (rest of the script)

# Se valida la existencia de las variables para mantener la consistencia de datos en los archivos JSON
# Ensure security_hotspot_data is also checked for existence
if 'metrics_data' in globals() and 'bug_data' in globals() and 'codesmell_data' in globals() and 'vuln_data' in globals() and security_hotspot_data is not None:

        # Si el archivo metrics.json ya existe... (no changes here)

    # Si el archivo key_data.json ya existe en el path correspondiente se agregan los datos nuevos en caso contrario se crea el archivo con los datos obtenidos
        if os.path.exists(key_data_file_path):
                with open(key_data_file_path, 'r') as file:
                        existing_data = json.load(file)
                        existing_data.append(bug_data)
                        existing_data.append(codesmell_data)
                        existing_data.append(vuln_data)
                        existing_data.append(security_hotspot_data) # ADD THIS LINE

                with open(key_data_file_path, 'w') as file:
                        json.dump(existing_data, file, indent=4)
                        print('El archivo key_data.json fue actualizado exitosamente.')

        else:
                with open(key_data_file_path,'w') as file:
                        # ADD security_hotspot_data to this list
                        key_data_list = [bug_data, codesmell_data, vuln_data, security_hotspot_data] 
                        json.dump(key_data_list, file, indent=4)
                        print('El archivo key_data.json fue creado exitosamente.')

else:
        # Update this message to reflect all data needed
        print('Hubo un error en la carga de datos. Revisar los llamados a la API de SonarQube')

# ... (rest of the script)
```

---

## Explanation:

By making these changes, your `sonarqube-report.py` script will now query SonarQube for `SECURITY_HOTSPOT` issues and include their counts (categorized by severity) directly into your `key_data.json`. This will provide a more accurate and comprehensive security overview for your PowerBI dashboards, reflecting the 65+ hotspots identified by SonarQube.

Remember that **Security Hotspots** are not automatically `VULNERABILITY` issues. Their classification still requires a manual review step within the SonarQube UI. However, including them in your report will give visibility to these pending security reviews.
