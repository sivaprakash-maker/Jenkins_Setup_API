
pipeline {

    agent any

    environment {
        VENV_DIR = 'venv'
        API_PORT = '5000'
    }

    options {
        disableConcurrentBuilds()
    }

    stages {

        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Set Up Python Environment') {
            steps {
                bat '''
                    echo ========================================
                    echo Python Environment Setup
                    echo ========================================

                    python --version
                    where python

                    echo.
                    echo Removing old virtual environment...

                    if exist "%VENV_DIR%" (
                        rmdir /s /q "%VENV_DIR%"
                    )

                    echo.
                    echo Creating virtual environment...

                    python -m venv "%VENV_DIR%"

                    if errorlevel 1 (
                        echo Failed to create virtual environment.
                        exit /b 1
                    )

                    echo.
                    echo Checking virtual environment Python...

                    "%VENV_DIR%\\Scripts\\python.exe" --version

                    echo.
                    echo Upgrading pip...

                    "%VENV_DIR%\\Scripts\\python.exe" -m pip install --upgrade pip

                    if errorlevel 1 (
                        echo Pip upgrade failed.
                        exit /b 1
                    )

                    echo.
                    echo Installing requirements...

                    "%VENV_DIR%\\Scripts\\python.exe" -m pip install -r requirements.txt

                    if errorlevel 1 (
                        echo Requirements installation failed.
                        exit /b 1
                    )

                    echo.
                    echo Testing NumPy...

                    "%VENV_DIR%\\Scripts\\python.exe" -c "import numpy; print('NumPy version:', numpy.__version__); print('NumPy location:', numpy.__file__)"

                    if errorlevel 1 (
                        echo NumPy test failed.
                        exit /b 1
                    )

                    echo.
                    echo Testing Pandas...

                    "%VENV_DIR%\\Scripts\\python.exe" -c "import pandas; print('Pandas version:', pandas.__version__); print('Pandas location:', pandas.__file__)"

                    if errorlevel 1 (
                        echo Pandas test failed.
                        exit /b 1
                    )

                    echo.
                    echo Testing Requests...

                    "%VENV_DIR%\\Scripts\\python.exe" -c "import requests; print('Requests version:', requests.__version__); print('Requests location:', requests.__file__)"

                    if errorlevel 1 (
                        echo Requests test failed.
                        exit /b 1
                    )

                    echo.
                    echo Python environment setup completed successfully.
                '''
            }
        }

        stage('Train Model') {
            steps {
                bat '''
                    echo ========================================
                    echo Training Model
                    echo ========================================

                    "%VENV_DIR%\\Scripts\\python.exe" train_model.py

                    if errorlevel 1 (
                        echo.
                        echo Model training failed.
                        exit /b 1
                    )

                    echo.
                    echo Model training completed successfully.

                    if not exist house_model.pkl (
                        echo ERROR: house_model.pkl was not created.
                        exit /b 1
                    )

                    echo Model file found successfully.
                '''
            }
        }

       
stage('Start API') {
    steps {
        bat '''
            echo ========================================
            echo Starting Flask API
            echo ========================================

            if exist app.log del /f /q app.log
            if exist flask.pid del /f /q flask.pid

            echo Starting API on port %API_PORT%...

            powershell -NoProfile -ExecutionPolicy Bypass -Command ^
              "$p = Start-Process -FilePath 'venv\\Scripts\\python.exe' -ArgumentList 'app.py' -RedirectStandardOutput 'app.log' -RedirectStandardError 'app.log' -PassThru; Set-Content -Path 'flask.pid' -Value $p.Id"

            echo API process started.
        '''
    }
}

stage('API Health Check') {
    steps {
        bat '''
            echo ========================================
            echo API Health Check
            echo ========================================

            powershell -NoProfile -ExecutionPolicy Bypass -Command "$ok=$false; for($i=1;$i -le 30;$i++){ try { $r=Invoke-WebRequest -Uri 'http://127.0.0.1:5000/' -UseBasicParsing -TimeoutSec 2; if($r.StatusCode -ge 200 -and $r.StatusCode -lt 500){ $ok=$true; Write-Host 'API is ready.'; break } } catch { }; Write-Host ('Waiting for API... {0}/30' -f $i); Start-Sleep -Seconds 1 }; if(-not $ok){ Write-Host 'API failed to start.'; Write-Host '===== app.log ====='; if(Test-Path 'app.log'){Get-Content 'app.log'}; Write-Host '==================='; exit 1 }"

            if errorlevel 1 (
                echo API health check failed.
                exit /b 1
            )

            echo API health check passed.
        '''
    }
}

stage('Prediction Smoke Test') {
    steps {
        bat '''
            echo ========================================
            echo Prediction Smoke Test
            echo ========================================

            "venv\\Scripts\\python.exe" test_prediction.py

            if errorlevel 1 (
                echo Prediction test failed.
                echo.
                echo ===== app.log =====
                if exist app.log type app.log
                echo ===================
                exit /b 1
            )

            echo.
            echo Prediction smoke test completed successfully.
        '''
    }
}

post {
    always {
        echo 'Cleaning up Flask API...'

        bat '''
            echo ========================================
            echo Cleaning API Process
            echo ========================================

            if exist flask.pid (
                echo Stopping Flask process...

                powershell -NoProfile -ExecutionPolicy Bypass -Command "$pid = Get-Content 'flask.pid' -ErrorAction SilentlyContinue; if($pid){ try { Stop-Process -Id ([int]$pid) -Force -ErrorAction SilentlyContinue; Write-Host 'Flask process stopped.' } catch { Write-Host 'Flask process already stopped.' } }"

                del /f /q flask.pid
            ) else (
                echo Flask PID file not found. Nothing to clean.
            )

            echo API cleanup completed.
        '''

        archiveArtifacts artifacts: 'house_model.pkl,app.log', allowEmptyArchive: true
    }

    success {
        echo '''
========================================
BUILD SUCCESSFUL
========================================
'''
    }

    failure {
        echo '''
========================================
BUILD FAILED
========================================
Check the console output and app.log.
========================================
'''
    }
}


        cleanup {

            bat '''
                echo ========================================
                echo Removing Virtual Environment
                echo ========================================

                if exist "%VENV_DIR%" (
                    rmdir /s /q "%VENV_DIR%"
                )

                echo Virtual environment cleanup completed.
            '''
        }
    }
}
