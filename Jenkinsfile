pipeline {
     agent { label 'devops-agent' }

    // --- 1. Dynamic Parameters ---
    // this allows Manual Overrides, but defaults to 'auto' for Webhooks
    parameters {
        choice(
            name: 'TARGET_ENV', 
            choices: ['auto', 'dev', 'qa', 'uat', 'prod'], 
            description: 'Select "auto" for webhook triggers, or override manually.'
        )
    }

    environment {
        // --- CREDENTIALS ---
        PROD_CLIENT_ID = credentials('c7f9a893-2e31-45b6-a06d-a9f20a9dc740')
        JWT_KEY_FILE   = credentials('73d31094-94d3-48cd-8aad-e4d832f909d5')
    }

    stages {
        // --- STAGE 1: Determine Environment & Checkout ---
        stage('Setup & Checkout') {
            steps {
                script {
                    // 1. Checkout Code
                    checkout scm
                    sh "git fetch --all" // Fetch all branches for Delta comparison

                    // 2. Determine Environment Logic
                    def selectedEnv = params.TARGET_ENV ?: 'auto'
                    def branchName = env.BRANCH_NAME
                    
                    // Logic: If 'auto', pick env based on Branch Name
                    if (selectedEnv == 'auto') {
                        if (branchName.startsWith('feature/')) {                            
                            env.FINAL_ENV = 'qa'  // Branch starts with feature (e.g., feature/us-001)-> Deploy to QA Org
                        } else if (branchName == 'integration') {
                            env.FINAL_ENV = 'uat'    // Branch is integration -> Deploy to UAT Org
                        } else if (branchName.startsWith('release/')) {
                            env.FINAL_ENV = 'prod'   // Branch starts with release (e.g., release/v1.0) -> Deploy to PROD Org
                        } else {
                            env.FINAL_ENV = 'dev' // Default fallback
                        }
                    } else {
                        env.FINAL_ENV = selectedEnv // Manual Override
                    }
                    
                    // 3. Set Org Alias & Credentials based on Final Env
                    env.SF_ALIAS = "${env.FINAL_ENV}_org"
                    
                    // Define usernames map
                    if (env.FINAL_ENV == 'qa') {
                        env.SF_USERNAME = "kduruvasula-demo@salesforce.com.qa"
                        env.SF_URL = "https://test.salesforce.com"
                    } else if (env.FINAL_ENV == 'uat') {
                        env.SF_USERNAME = "kduruvasula-demo@salesforce.com.uat"
                        env.SF_URL = "https://login.salesforce.com"
                    } else if (env.FINAL_ENV == 'prod') {
                        env.SF_USERNAME = "kduruvasula-demo@salesforce.com"
                        env.SF_URL = "https://login.salesforce.com"
                        env.SF_ALIAS = "prod_org"
                    } else {
                        env.SF_USERNAME = "kduruvasula-demo@salesforce.com.dev1"
                        env.SF_URL = "https://test.salesforce.com"
                    }

                    echo "------------------------------------------------"
                    echo "Branch: ${branchName} | Target: ${env.FINAL_ENV}"
                    echo "------------------------------------------------"
                } //End of Script
            } //End of Steps
        } // End of Setup & Checkout stage

        // --- STAGE 2: Authenticate ---
        stage('Authenticate') {
            steps {
                script {
                    sh """
                        sf org login jwt \
                            --client-id "${env.PROD_CLIENT_ID}" \
                            --jwt-key-file "${env.JWT_KEY_FILE}" \
                            --username "${env.SF_USERNAME}" \
                            --instance-url "${env.SF_URL}" \
                            --alias "${env.SF_ALIAS}" \
                            --set-default
                    """
                } //End of Script
            } //End of Steps
        } // End of Authenticate stage

        // --- STAGE 3: Generate Delta (DYNAMIC LOGIC ADDED HERE) ---
        stage('Generate Delta') {
            steps {
                script {
                    echo "Cleaning previous delta artifacts..."  
                    sh "rm -rf changed-sources"
                    sh "mkdir -p changed-sources"
                    sh "echo 'y' | sf plugins install sfdx-git-delta || true"

                    def currentBranch = env.BRANCH_NAME
                    def compareToBranch = "main" // Default fallback

                    // Define your specific rules:
                    if (currentBranch.startsWith('feature/')) {
                        // Feature branches compare against Integration
                        compareToBranch = "integration"
                    } else if (currentBranch == 'integration') {
                        // Integration compares against Release
                        compareToBranch = "release" 
                    } else if (currentBranch.startsWith('release/')) {
                        // Release compares against Main
                        compareToBranch = "main"
                    }

                    echo "Current Branch: ${currentBranch}"
                    echo "Calculating Delta against: origin/${compareToBranch}"

                    // Ensure the comparison branch actually exists in our fetch
                    sh "git fetch origin ${compareToBranch}:refs/remotes/origin/${compareToBranch} || true"
                    
                    // Run Delta with Dynamic Branch
                    sh """
                        sf sgd:source:delta \
                        --to "HEAD" \
                        --from "origin/${compareToBranch}" \
                        --output-dir "changed-sources" \
                        --generate-delta
                    """
                }  //End of Script
            } //End of Steps
        } // End of Generate Delta stage
        // --- STAGE 4: Validate & Test (Apex Tests) ---
        stage('Validate & Apex Tests') {
            steps {
                script {
                    if (fileExists('changed-sources/package/package.xml')) {
                        echo "Validating and Running Local Tests on ${env.SF_ALIAS}..."
                        
                        // --dry-run: Don't save yet
                        // --test-level RunLocalTests: Runs all local tests in the org to ensure quality
                        sh """
                            sf project deploy start \
                                --dry-run \
                                --manifest "changed-sources/package/package.xml" \
                                --target-org "${env.SF_ALIAS}" \
                                --test-level RunLocalTests \
                                --wait 30
                        """
                    } else {
                        echo "No changes to validate."
                    }
                } //End of Script
            } //End of Steps
        } // End of Validate & Test stage

        // --- STAGE 5: Deploy Changes ---
        stage('Deploy Changes') {
            steps {
                script {
                    //deploy command base
                    def deployCommand = "sf project deploy start --manifest 'changed-sources/package/package.xml' --target-org '${env.SF_ALIAS}' --ignore-conflicts --wait 30"
                    
                    // Check if there are DELETIONS (destructiveChanges.xml)
                    if (fileExists('changed-sources/destructiveChanges/destructiveChanges.xml')) {
                        echo "Destructive Changes Detected!"
                        // Append the flag to apply deletions AFTER the deployment
                        deployCommand += " --post-destructive-changes 'changed-sources/destructiveChanges/destructiveChanges.xml'"
                    }

                    // Check if there are ADDITIONS (package.xml) or DELETIONS to process
                    if (fileExists('changed-sources/package/package.xml') || fileExists('changed-sources/destructiveChanges/destructiveChanges.xml')) {
                        echo "Deploying to ${env.SF_ALIAS}..."
                        sh "${deployCommand}"
                    } else {
                        echo "No changes (additions or deletions) to deploy."
                    }
                } //script
            } //steps
        } //End of Deploy Changes stage
    } // End of Stages
} // End of Pipeline