pipeline {
    agent { label 'master' }
    stages {
        stage('Process') {
      steps {
        script {
          env.INPUT_FILE = "${WORKSPACE}/input.csv"

          sh """
#!/bin/bash

# Decode the Base64 content and save to file
echo "INPUT_FILE_PATH" | base64 --decode > "INPUT_FILE"

# Verify the file creation
if [ -f "$INPUT_FILE" ]; then
    echo "File has been successfully decoded and saved to $INPUT_FILE"
else
    echo "Failed to decode and save the file."
fi
                    """

          echo "File saved to ${filePath}"

          // Archive the file as an artifact
          archiveArtifacts artifacts: 'input.csv', allowEmptyArchive: true
        }
      }
        }
    }
}