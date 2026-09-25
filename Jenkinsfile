
pipeline {
    agent any

    environment {
        VENV_DIR = 'venv'
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

                    if exist "%VENV_DIR%" (
                        echo Removing old virtual environment...
                        rmdir /s /q "%VENV_DIR%"
                    )

                    echo Creating virtual environment...
                    python -m venv "%VENV_DIR%"

                    echo Checking virtual environment Python...
                    "%VENV_DIR%\\Scripts\\python.exe" --version

                    echo Upgrading pip...
                    "%VENV_DIR%\\Scripts\\python.exe" -m pip install --upgrade pip

                    echo Installing requirements...
                    "%VENV_DIR%\\Scripts\\python.exe" -m pip install -r requirements.txt

                    echo Testing NumPy...
                    "%VENV_DIR%\\Scripts\\python.exe" -c "import numpy; print('NumPy version:', numpy.__version__); print('NumPy location:', numpy.__file__)"

                    echo Testing Pandas...
                    "%VENV_DIR%\\Scripts\\python.exe" -c "import pandas; print('Pandas version:', pandas.__version__); print('Pandas location:', pandas.__file__)"

                    echo Python environment setup completed.
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
                        echo Model training failed.
                        exit /b 1
                    )

                    echo Model training completed successfully.
                '''
            }
        }

        stage('Start API & Smoke Test') {
            steps {
                bat '''
                    echo ========================================
                    echo Starting API
                    echo ========================================

                    if exist app.log del /f /q app.log

                    start "FlaskAPI" /B cmd /c ""%VENV_DIR%\\Scripts\\python.exe" app.py > app.log 2>&1"

                    echo Waiting for API...

                    set READY=0

                    for /L %%i in (1,1,30) do (

                        powershell -NoProfile -ExecutionPolicy Bypass -Command "try { $r = Invoke-WebRequest -Uri 'http://127.0.0.1:5000/' -UseBasicParsing -TimeoutSec 2; if ($r.StatusCode -ge 200 -and $r.StatusCode -lt 500) { exit 0 } else { exit 1 } } catch { exit 1 }"

                        if not errorlevel 1 (
                            set READY=1
                            echo API is ready.
                            goto API_READY
                        )

                        echo Waiting... %%i/30
                        timeout /t 1 /nobreak >nul
                    )

                    :API_READY

                    if "%READY%"=="0" (
                        echo API failed to start.
                        echo.
                        echo ===== app.log =====

                        if exist app.log (
                            type app.log
                        )

                        echo ===================
                        exit /b 1
                    )

                    echo Running prediction test...

                    "%VENV_DIR%\\Scripts\\python.exe" test_prediction.py

                    if errorlevel 1 (
                        echo Prediction test failed.
                        exit /b 1
                    )

                    echo Smoke test completed successfully.
                '''
            }
        }
    }

    post {

        always {
            bat '''
                echo ========================================
                echo Cleaning API process
                echo ========================================

                powershell -NoProfile -ExecutionPolicy Bypass -Command "Get-CimInstance Win32_Process | Where-Object { $_.CommandLine -like '*app.py*' } | ForEach-Object { Stop-Process -Id $_.ProcessId -Force -ErrorAction SilentlyContinue }"

                echo API cleanup completed.
            '''

            archiveArtifacts artifacts: 'house_model.pkl, app.log',
                             allowEmptyArchive: true
        }

        success {
            echo 'Build, train, and smoke test succeeded.'
        }

        failure {
            echo 'Pipeline failed. Check the console output and app.log.'
        }

        cleanup {
            bat '''
                echo Removing virtual environment...

                if exist "%VENV_DIR%" (
                    rmdir /s /q "%VENV_DIR%"
                )

                echo Cleanup completed.
            '''
        }
    }
}
