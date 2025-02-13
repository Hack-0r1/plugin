#first
 #winget install --id GitHub.cli
# gh auth login
# winget install --id Microsoft.Powershell --source winget


$repo_owner = "wp-plugins"
$batch_size = 100  # Reduce batch size to avoid memory issues
$max_parallel_jobs = 10  # Reduce parallel jobs to prevent stack overflow
$clone_path = "C:\Users\DBMS\music\plugins"  # Set your directory
$log_file = "$clone_path\clone_log.txt"

# Ensure the directory exists
if (!(Test-Path -Path $clone_path)) {
    New-Item -ItemType Directory -Path $clone_path | Out-Null
}

Set-Location -Path $clone_path  # Navigate to the target directory
Write-Host "Fetching repository list from $repo_owner..." | Tee-Object -FilePath $log_file -Append

$total_repos = 0
$page = 200  # Start pagination manually
$jobList = @()  # Correct way to initialize a job list

while ($true) {
    Write-Host "📥 Fetching repositories from page $page..." | Tee-Object -FilePath $log_file -Append

    # Fetch repositories with controlled pagination
    $repos = gh api "orgs/$repo_owner/repos?per_page=$batch_size&page=$page" --jq ".[].name" 2>&1

    if ($repos.Count -eq 0) {
        Write-Host "✅ No more repositories left to fetch. Stopping." | Tee-Object -FilePath $log_file -Append
        break
    }

    Write-Host "🔍 Found $($repos.Count) repositories on page $page" | Tee-Object -FilePath $log_file -Append

    foreach ($repo in $repos) {
        $repoPath = "$clone_path\$repo"

        # Skip if the repository already exists
        if (Test-Path -Path $repoPath) {
            Write-Host "⏭️ Skipping already cloned repository: $repo" | Tee-Object -FilePath $log_file -Append
            continue
        }

        # Start a background job for cloning
        Write-Host "🚀 Cloning repository: $repo" | Tee-Object -FilePath $log_file -Append
        $jobList += @(Start-Job -ScriptBlock {
            param ($repo, $repo_owner, $clone_path, $log_file)
            Set-Location -Path $clone_path
            $output = gh repo clone "$repo_owner/$repo" 2>&1
            $output | Tee-Object -FilePath $log_file -Append
        } -ArgumentList $repo, $repo_owner, $clone_path, $log_file)

        $total_repos++

        # Limit parallel cloning jobs
        while (($jobList | Where-Object { $_.State -eq 'Running' }).Count -ge $max_parallel_jobs) {
            Start-Sleep -Seconds 5
        }

        # Show progress every 50 repos
        if ($total_repos % 50 -eq 0) {
            Write-Host "📊 Processed $total_repos repositories so far..." | Tee-Object -FilePath $log_file -Append
        }
    }

    # Wait for all background jobs to finish before fetching the next page
    Write-Host "⏳ Waiting for all cloning jobs to complete..." | Tee-Object -FilePath $log_file -Append
    Get-Job | Wait-Job | Out-Null
    Get-Job | Remove-Job

    # Move to the next page
    $page++

    # Stop if fewer than batch_size repositories were returned (end of results)
    if ($repos.Count -lt $batch_size) {
        Write-Host "✅ Last batch received. Stopping..." | Tee-Object -FilePath $log_file -Append
        break
    }
}

Write-Host "✅ Cloning complete! Total repositories processed: $total_repos" | Tee-Object -FilePath $log_file -Append
