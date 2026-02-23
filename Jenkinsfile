pipeline {
    agent any
    options { timestamps() }

    stages {
        stage('Get Code') {
            steps {
                checkout scm
                sh 'echo "Branch: $BRANCH_NAME"'
                sh 'git rev-parse --short HEAD'
                sh 'test -d src'  // falla rápido si no existe /src
            }
        }

        stage('Pruebas de código estáticas') {
            steps {
                sh '''
          set +e

          python3 -m venv .venv
          . .venv/bin/activate
          pip install -U pip
          pip install flake8 bandit

          mkdir -p reports

          echo "Running flake8 (src only)..."
          flake8 src --count --statistics --exit-zero | tee reports/flake8.txt

          echo "Running bandit (src only)..."
          bandit -r src -f txt -o reports/bandit.txt || true

          echo "Static analysis done. Reports generated in ./reports"
        '''
                archiveArtifacts artifacts: 'reports/*', fingerprint: true
            }
        }
    }
}
