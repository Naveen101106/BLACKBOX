from flask import Flask, jsonify, request
from flask_cors import CORS
import os
from datetime import datetime

app = Flask(__name__)
CORS(app)  # Enable CORS for the entire application

TARGET_DIR = "/etc/alloy"  # Directory to store Prometheus configuration files

# Endpoint to add a Prometheus blackbox configuration
@app.route('/endpoint', methods=['POST'])
def add_prometheus_record():
    """Insert a new record into the Prometheus configuration and create config files."""
    try:
        data = request.get_json()
        name = data.get('name')
        url = data.get('url')

        # Ensure required fields are provided
        if not name or not url:
            return jsonify({"error": "Missing required fields (name, url)"}), 400

        config_file_path = os.path.join(TARGET_DIR, f"{name}.yml")

        # Check if configuration for this name already exists
        if os.path.exists(config_file_path):
            return jsonify({
                "error": f"Configuration for '{name}' already exists. Duplicate configurations are not allowed."
            }), 409

        os.makedirs(TARGET_DIR, exist_ok=True)

        # Generate blackbox configuration
        blackbox_config = f"""
prometheus.exporter.blackbox "{name}" {{
  config_file = "{config_file_path}"
  target {{
    name    = "{name}"
    address = "{url}"
    module  = "http_2xx"
    labels = {{
      "sitename" = "{name}"
    }}
  }}
}}
prometheus.scrape "{name}_demo" {{
  targets    = prometheus.exporter.blackbox.{name}.targets
  forward_to = [prometheus.remote_write.{name}_demo.receiver]
}}
prometheus.remote_write "{name}_demo" {{
  endpoint {{
    url = "http://localhost:9090/api/v1/write"
  }}
}}
"""

        # Write the configuration file
        try:
            with open(config_file_path, "w") as config_file:
                config_file.write(blackbox_config)
        except Exception as e:
            return jsonify({"error": "Failed to create configuration file"}), 500

        return jsonify({"message": f"Prometheus record '{name}' added successfully"}), 201

    except Exception as e:
        return jsonify({"error": "Failed to handle the request"}), 500


# Endpoint to delete a Prometheus configuration by name
@app.route('/endpoint/delete', methods=['DELETE'])
def delete_prometheus_record():
    """Delete a Prometheus configuration based on its 'name'."""
    try:
        data = request.get_json()  # Parse JSON from the request body
        name = data.get('name')

        if not name:
            return jsonify({"error": "Missing 'name' in request body"}), 400

        config_file_path = os.path.join(TARGET_DIR, f"{name}.yml")

        # Check if configuration file exists
        if not os.path.exists(config_file_path):
            return jsonify({"error": f"No configuration found for name '{name}'"}), 404

        # Delete the configuration file
        try:
            os.remove(config_file_path)
        except Exception as e:
            return jsonify({"error": f"Failed to delete configuration file for '{name}'"}), 500

        return jsonify({"message": f"Configuration for '{name}' deleted successfully"}), 200

    except Exception as e:
        return jsonify({"error": "Failed to handle the DELETE request"}), 500


if __name__ == '__main__':
    app.run(debug=True, host='0.0.0.0', port=5000)
