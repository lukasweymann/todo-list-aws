pipeline {
    agent any
    options {
        timestamps()
        disableConcurrentBuilds()
    }

    environment {
        AWS_REGION = 'eu-west-1'
        AWS_DEFAULT_REGION = 'eu-west-1'
        // Credenciales de github del entorno de Jenkins
        GIT_WRITE_CRED_ID = 'github-pat-write'
        // key de cloud formation para poder hacer pytest
        BASE_URL_OUTPUT_KEY = 'ApiUrl'
    }

    stages {
        stage('Get Code') {
            steps {
                checkout scm
                sh 'echo "Branch: $BRANCH_NAME"'
                sh 'git rev-parse --short HEAD'
                sh 'test -d src'
            }
        }

        stage('Static Test ') {
            steps {
                sh '''
          set +e
          python3 -m venv .venv
          . .venv/bin/activate

          # Pipeline determinístico, puede que estén las dependencias en la máquina pero optamos por reproducibilidad en cualquier máquina/agente

          pip install -U pip
          pip install flake8 bandit

          mkdir -p reports

          echo "Tests con flake8 sólo en directorio src como indicado"
          flake8 src --count --statistics --exit-zero | tee reports/flake8.txt

          echo "Tests con bandit..."
          bandit -r src -f txt -o reports/bandit.txt || true
        '''
                archiveArtifacts artifacts: 'reports/*', fingerprint: true
            }
        }
        stage('Deploy (SAM) - Staging') {
            steps {
                sh '''
      set -e
      sam --version
      aws --version

      echo "Validando..."
      sam validate --region "$AWS_REGION"

      echo "Construyendo..."
      sam build

      echo "Deploy using samconfig.toml (staging env)..."
      sam deploy --config-env staging  --resolve-s3\
      --no-confirm-changeset \
      --no-fail-on-empty-changeset

    '''
            }
        }
        stage('Rest Test') {
            steps {
                sh '''
          set -e
          . .venv/bin/activate || true

          python3 -m venv .venv
          . .venv/bin/activate
          pip install -U pip
          pip install pytest requests

          echo "Extracción de la URL de cloudformation..."
          BASE_URL=$(aws cloudformation describe-stacks \
                                --stack-name todo-list-aws-staging \
                                --region us-east-1 \
                                --query "Stacks[0].Outputs[?OutputKey=='BaseUrlApi'].OutputValue" \
                                --output text)

          if [ -z "$BASE_URL" ] || [ "$BASE_URL" = "None" ]; then
            echo "ERROR: no se ha podido leer la url desde '$BASE_URL_OUTPUT_KEY'."
            exit 1
          fi

          echo "BASE_URL=$BASE_URL"

          # Haciendo la URL accesible para pytest
          export BASE_URL="$BASE_URL"

          echo "Tests de integración..."
          pytest -q test/integration/todoApiTest.py
        '''
            }
        }

        stage('Promote') {
            when { branch 'develop' }
            steps {
                sh '''
          set -e
          echo "Release build marker" > RELEASE.txt
          echo "Commit: $(git rev-parse HEAD)" >> RELEASE.txt
          date -Is >> RELEASE.txt
        '''
                archiveArtifacts artifacts: 'RELEASE.txt', fingerprint: true

                // Merge develop -> master + push (requires write credential)
                withCredentials([usernamePassword(
          credentialsId: "${GIT_WRITE_CRED_ID}",
          usernameVariable: 'GH_USER',
          passwordVariable: 'GH_TOKEN'
        )]) {
                    sh '''
            set -e
            git config user.email "ci-bot@example.com"
            git config user.name "ci-bot"

            git fetch origin --prune '+refs/heads/*:refs/remotes/origin/*'

            # Asegurar que el merge se haga con el commit en concreto para evitar inconsistencia
            TESTED_COMMIT=$(git rev-parse HEAD)

            git checkout master
            git reset --hard origin/master

            # Mergeo de la rama develop a master
            git merge --no-ff "$TESTED_COMMIT" -m "Promote: release from develop ($TESTED_COMMIT)"

            # Subir cambios sin exponer datos sensibles en echo
            git push "https://${GH_USER}:${GH_TOKEN}@github.com/lukasweymann/todo-list-aws.git" master
          '''
        }
            }
        }
    }
}
