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
// pipeline {
//     agent any
//     options {
//         timestamps()
//         disableConcurrentBuilds()
//     }

//     environment {
//         // Single, consistent region everywhere
//         AWS_REGION = 'eu-west-1'
//         AWS_DEFAULT_REGION = 'eu-west-1'

//         // GitHub write credential for promote (develop -> master)
//         GIT_WRITE_CRED_ID = 'github-pat-write'

//         // CloudFormation stack + output key for API base URL
//         STAGING_STACK = 'todo-list-aws-staging'
//         BASE_URL_OUTPUT_KEY = 'ApiUrl'
//     }

//     stages {
//         stage('Get Code') {
//             steps {
//                 checkout scm
//                 sh '''
//                   set -e
//                   echo "Branch: $BRANCH_NAME"
//                   git rev-parse --short HEAD
//                   test -d src
//                 '''
//             }
//         }

//         stage('Static Test') {
//             steps {
//                 sh '''
//                   set -e

//                   # Deterministic pipeline: use venv for reproducibility across agents
//                   if [ ! -d ".venv" ]; then
//                     python3 -m venv .venv
//                   fi
//                   . .venv/bin/activate

//                   pip install -U pip
//                   pip install flake8 bandit

//                   mkdir -p reports

//                   echo "flake8 (src only)"
//                   flake8 src --count --statistics --exit-zero | tee reports/flake8.txt

//                   echo "bandit (src only)"
//                   bandit -r src -f txt -o reports/bandit.txt || true
//                 '''
//                 archiveArtifacts artifacts: 'reports/*', fingerprint: true
//             }
//         }

//         stage('Deploy (SAM) - Staging') {
//             steps {
//                 sh '''
//                   set -e
//                   sam --version
//                   aws --version

//                   echo "Validate..."
//                   sam validate --region "$AWS_REGION"

//                   echo "Build..."
//                   sam build

//                   echo "Deploy (staging) using samconfig.toml..."
//                   sam deploy \
//                     --config-env staging \
//                     --region "$AWS_REGION" \
//                     --resolve-s3 \
//                     --no-confirm-changeset \
//                     --no-fail-on-empty-changeset
//                 '''
//             }
//         }

//         stage('Rest Test (Pytest)') {
//             steps {
//                 sh '''
//                   set -e

//                   # Reuse venv from Static Test; create if missing
//                   if [ ! -d ".venv" ]; then
//                     python3 -m venv .venv
//                   fi
//                   . .venv/bin/activate

//                   pip install -U pip
//                   pip install pytest requests

//                   mkdir -p reports

//                   echo "Reading BASE_URL from CloudFormation (stack=$STAGING_STACK, region=$AWS_REGION, OutputKey=$BASE_URL_OUTPUT_KEY)..."
//                   BASE_URL=$(aws cloudformation describe-stacks \
//                     --stack-name "$STAGING_STACK" \
//                     --region "$AWS_REGION" \
//                     --query "Stacks[0].Outputs[?OutputKey=='$BASE_URL_OUTPUT_KEY'].OutputValue" \
//                     --output text)

//                   if [ -z "$BASE_URL" ] || [ "$BASE_URL" = "None" ]; then
//                     echo "ERROR: Could not read API URL from OutputKey='$BASE_URL_OUTPUT_KEY'."
//                     echo "Available stack outputs:"
//                     aws cloudformation describe-stacks \
//                       --stack-name "$STAGING_STACK" \
//                       --region "$AWS_REGION" \
//                       --query "Stacks[0].Outputs" \
//                       --output table || true
//                     exit 1
//                   fi

//                   echo "BASE_URL=$BASE_URL"
//                   export BASE_URL="$BASE_URL"

//                   echo "Running integration tests with pytest (verbose for evidence in logs)..."
//                   pytest -vv test/integration/todoApiTest.py \
//                     --junitxml=reports/pytest-junit.xml | tee reports/pytest-output.txt
//                 '''
//                 archiveArtifacts artifacts: 'reports/*', fingerprint: true
//             }
//         }

//         stage('Promote') {
//             when { branch 'develop' }
//             steps {
//                 sh '''
//                   set -e
//                   echo "Release build marker" > RELEASE.txt
//                   echo "Commit: $(git rev-parse HEAD)" >> RELEASE.txt
//                   date -Is >> RELEASE.txt
//                 '''
//                 archiveArtifacts artifacts: 'RELEASE.txt', fingerprint: true

//                 withCredentials([usernamePassword(
//                     credentialsId: "${GIT_WRITE_CRED_ID}",
//                     usernameVariable: 'GH_USER',
//                     passwordVariable: 'GH_TOKEN'
//                 )]) {
//                     sh '''
//                       set -e
//                       git config user.email "ci-bot@example.com"
//                       git config user.name "ci-bot"

//                       # Ensure all remote branches are available in multibranch workspaces
//                       git fetch origin --prune '+refs/heads/*:refs/remotes/origin/*'

//                       # Tested commit (current HEAD of develop build)
//                       TESTED_COMMIT=$(git rev-parse HEAD)

//                       # Robustly reset local master from origin/master
//                       git checkout -B master origin/master

//                       # Merge develop commit into master (no fast-forward for traceability)
//                       git merge --no-ff "$TESTED_COMMIT" -m "Promote: release from develop ($TESTED_COMMIT)"

//                       # Push using token (do NOT echo token)
//                       git push "https://${GH_USER}:${GH_TOKEN}@github.com/lukasweymann/todo-list-aws.git" master
//                     '''
//                 }
//             }
//         }
//     }
// }
