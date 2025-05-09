# PROJECT-MAHA
MAHA=Machine Automated Healthcare Assistent

AI for Social Good: A Drug Discovery Project

this project has been deflected in some issue in core program of some issue to make over 

name: AI Bot

on:
  issues:
    types: [opened]
  issue_comment:
    types: [created]
  pull_request_review_comment:
    types: [created]
  pull_request:
    types: [opened, edited, synchronize]

jobs:
  respond-to-commands:
    runs-on: ubuntu-latest
    if: |
      (github.actor == 'f') &&
      ((github.event_name == 'issues' && contains(github.event.issue.body, '/ai')) ||
      (github.event_name == 'issue_comment' && contains(github.event.comment.body, '/ai')) ||
      (github.event_name == 'pull_request_review_comment' && contains(github.event.comment.body, '/ai')) ||
      (github.event_name == 'pull_request' && contains(github.event.pull_request.body, '/ai')))
    permissions:
      contents: write
      pull-requests: write
      issues: write

    steps:
      - uses: actions/checkout@v3
        with:
          fetch-depth: 0
          token: ${{ secrets.PAT_TOKEN }}

      - name: Setup Node.js
        uses: actions/setup-node@v3
        with:
          node-version: "18"

      - name: Install dependencies
        run: npm install openai@^4.0.0 @octokit/rest@^19.0.0

      - name: Process command
        id: process
        env:
          GH_TOKEN: ${{ secrets.PAT_TOKEN }}
          OPENAI_API_KEY: ${{ secrets.OPENAI_API_KEY }}
        run: |
          node << 'EOF'
          const OpenAI = require('openai');
          const { Octokit } = require('@octokit/rest');

          async function main() {
            const openai = new OpenAI({
              apiKey: process.env.OPENAI_API_KEY
            });

            const octokit = new Octokit({
              auth: process.env.GH_TOKEN
            });

            const eventName = process.env.GITHUB_EVENT_NAME;
            const eventPath = process.env.GITHUB_EVENT_PATH;
            const event = require(eventPath);

            // Double check user authorization
            const actor = event.sender?.login || event.pull_request?.user?.login || event.issue?.user?.login;
            if (actor !== 'f') {
              console.log('Unauthorized user attempted to use the bot:', actor);
              return;
            }

            // Get command and context
            let command = '';
            let issueNumber = null;
            let isPullRequest = false;

            if (eventName === 'issues') {
              command = event.issue.body;
              issueNumber = event.issue.number;
            } else if (eventName === 'issue_comment') {
              command = event.comment.body;
              issueNumber = event.issue.number;
              isPullRequest = !!event.issue.pull_request;
            } else if (eventName === 'pull_request_review_comment') {
              command = event.comment.body;
              issueNumber = event.pull_request.number;
              isPullRequest = true;
            } else if (eventName === 'pull_request') {
              command = event.pull_request.body;
              issueNumber = event.pull_request.number;
              isPullRequest = true;
            }

            if (!command.startsWith('/ai')) {
              return;
            }

            // Extract the actual command after /ai
            const aiCommand = command.substring(3).trim();

            // Handle resolve conflicts command
            if (aiCommand === 'resolve' || aiCommand === 'fix conflicts') {
              if (!isPullRequest) {
                console.log('Command rejected: Not a pull request');
                await octokit.issues.createComment({
                  owner: event.repository.owner.login,
                  repo: event.repository.name,
                  issue_number: issueNumber,
                  body: '❌ The resolve command can only be used on pull requests.'
                });
                return;
              }

              try {
                console.log('Starting resolve command execution...');

                // Get PR details
                console.log('Fetching PR details...');
                const { data: pr } = await octokit.pulls.get({
                  owner: event.repository.owner.login,
                  repo: event.repository.name,
                  pull_number: issueNumber
                });
                console.log(`Original PR found: #${issueNumber} from ${pr.user.login}`);

                // Get the PR diff to extract the new prompt
                console.log('Fetching PR file changes...');
                const { data: files } = await octokit.pulls.listFiles({
                  owner: event.repository.owner.login,
                  repo: event.repository.name,
                  pull_number: issueNumber
                });
                console.log(`Found ${files.length} changed files`);

                // Extract prompt from changes
                console.log('Analyzing changes to extract prompt information...');
                const prompts = new Map(); // Use Map to deduplicate prompts by title

                // Helper function to normalize prompt titles
                const normalizeTitle = (title) => {
                  title = title.trim();
                  // Remove "Act as" or "Act as a" or "Act as an" from start if present
                  title = title.replace(/^Act as (?:a |an )?/i, '');

                  // Capitalize each word except common articles and prepositions
                  const lowercaseWords = ['a', 'an', 'the', 'and', 'but', 'or', 'for', 'nor', 'on', 'at', 'to', 'for', 'with', 'in'];
                  const capitalized = title.toLowerCase().split(' ').map((word, index) => {
                    // Always capitalize first and last word
                    if (index === 0 || !lowercaseWords.includes(word)) {
                      return word.charAt(0).toUpperCase() + word.slice(1);
                    }
                    return word;
                  }).join(' ');

                  // Add "Act as" prefix
                  return `Act as ${capitalized}`;
                };

                // First try to find prompts in README
                let foundInReadme = false;
                for (const file of files) {
                  console.log(`Processing file: ${file.filename}`);
                  if (file.filename === 'README.md') {
                    const patch = file.patch || '';
                    const addedLines = patch.split('\n')
                      .filter(line => line.startsWith('+'))
                      .map(line => line.substring(1))
                      .join('\n');

                    console.log('Attempting to extract prompts from README changes...');
                    const promptMatches = [...addedLines.matchAll(/## (?:Act as (?:a |an )?)?([^\n]+)\n(?:Contributed by:[^\n]*\n)?(?:> )?([^#]+?)(?=\n##|\n\n##|$)/ig)];

                    for (const match of promptMatches) {
                      const actName = normalizeTitle(match[1]);
                      const promptText = match[2].trim()
                        .replace(/^(?:Contributed by:?[^\n]*\n\s*)+/i, '')
                        .trim();
                      const contributorLine = addedLines.match(/Contributed by: \[@([^\]]+)\]\(https:\/\/github\.com\/([^\)]+)\)/);
                      const contributorInfo = contributorLine
                        ? `Contributed by: [@${contributorLine[1]}](https://github.com/${contributorLine[2]})`
                        : `Contributed by: [@${pr.user.login}](https://github.com/${pr.user.login})`;

                      prompts.set(actName.toLowerCase(), { actName, promptText, contributorInfo });
                      console.log(`Found prompt in README: "${actName}"`);
                      foundInReadme = true;
                    }
                  }
                }

                // Only look in CSV if we didn't find anything in README
                if (!foundInReadme) {
                  console.log('No prompts found in README, checking CSV...');
                  for (const file of files) {
                    if (file.filename === 'prompts.csv') {
                      const patch = file.patch || '';
                      const addedLines = patch.split('\n')
                        .filter(line => line.startsWith('+'))
                        .map(line => line.substring(1))
                        .filter(line => line.trim()); // Remove empty lines

                      console.log('Attempting to extract prompts from CSV changes...');
                      for (const line of addedLines) {
                        // Parse CSV line considering escaped quotes
                        const matches = [...line.matchAll(/"([^"]*(?:""[^"]*)*)"/g)];
                        if (matches.length >= 2) {
                          const actName = normalizeTitle(matches[0][1].replace(/""/g, '"').trim());
                          const promptText = matches[1][1].replace(/""/g, '"').trim()
                            .replace(/^(?:Contributed by:?[^\n]*\n\s*)+/i, '')
                            .trim();

                          const contributorInfo = `Contributed by: [@${pr.user.login}](https://github.com/${pr.user.login})`;
                          prompts.set(actName.toLowerCase(), { actName, promptText, contributorInfo });
                          console.log(`Found prompt in CSV: "${actName}"`);
                        }
                      }
                    }
                  }
                }

                if (prompts.size === 0) {
                  console.log('Failed to extract prompt information');
                  await octokit.issues.createComment({
                    owner: event.repository.owner.login,
                    repo: event.repository.name,
                    issue_number: issueNumber,
                    body: '❌ Could not extract prompt information from changes'
                  });
                  return;
                }

                // Get content from main branch
                console.log('Fetching current content from main branch...');
                const { data: readmeFile } = await octokit.repos.getContent({
                  owner: event.repository.owner.login,
                  repo: event.repository.name,
                  path: 'README.md',
                  ref: 'main'
                });

                const { data: csvFile } = await octokit.repos.getContent({
                  owner: event.repository.owner.login,
                  repo: event.repository.name,
                  path: 'prompts.csv',
                  ref: 'main'
                });

                // Prepare new content
                console.log('Preparing content updates...');
                let readmeContent = Buffer.from(readmeFile.content, 'base64').toString('utf-8');
                let csvContent = Buffer.from(csvFile.content, 'base64').toString('utf-8');
                if (!csvContent.endsWith('\n')) csvContent += '\n';

                // Convert Map to array for processing
                const promptsArray = Array.from(prompts.values());

                // Process each prompt
                for (const { actName, promptText, contributorInfo } of promptsArray) {
                  // Remove markdown quote character and trim whitespace
                  const cleanPrompt = promptText.replace(/^>\s*/gm, '').trim();

                  // For README: Add quote to each line
                  const readmePrompt = cleanPrompt.split('\n')
                    .map(line => `> ${line.trim()}`)
                    .join('\n');
                  const newSection = `## ${actName}\n${contributorInfo}\n\n${readmePrompt}\n\n`;

                  // For CSV: Convert to single paragraph
                  const csvPrompt = cleanPrompt.replace(/\n+/g, ' ').trim();

                  // Insert the new section before Contributors in README
                  const contributorsIndex = readmeContent.indexOf('## Contributors');
                  if (contributorsIndex === -1) {
                    console.log('Contributors section not found, appending to end');
                    readmeContent += newSection;
                  } else {
                    console.log('Inserting before Contributors section');
                    readmeContent = readmeContent.slice(0, contributorsIndex) + newSection + readmeContent.slice(contributorsIndex);
                  }

                  // Add to CSV content
                  csvContent += `"${actName.replace(/^Act as an?/i, '').replace(/"/g, '""')}","${csvPrompt.replace(/"/g, '""')}"\n`;
                }

                // Create new branch
                const branchName = `prompt/${promptsArray.map(p => p.actName.toLowerCase().replace(/[^a-z0-9]+/g, '-')).join('-')}`;
                console.log(`Creating new branch: ${branchName}`);

                // Check if branch exists and delete it
                try {
                  console.log('Checking if branch already exists...');
                  const { data: existingRef } = await octokit.git.getRef({
                    owner: event.repository.owner.login,
                    repo: event.repository.name,
                    ref: `heads/${branchName}`
                  });

                  if (existingRef) {
                    // Check for existing PRs from this branch
                    console.log('Checking for existing PRs from this branch...');
                    const { data: existingPRs } = await octokit.pulls.list({
                      owner: event.repository.owner.login,
                      repo: event.repository.name,
                      head: `${event.repository.owner.login}:${branchName}`,
                      state: 'open'
                    });

                    // Close any existing PRs
                    for (const pr of existingPRs) {
                      console.log(`Closing existing PR #${pr.number}...`);
                      await octokit.pulls.update({
                        owner: event.repository.owner.login,
                        repo: event.repository.name,
                        pull_number: pr.number,
                        state: 'closed'
                      });
                    }

                    console.log('Branch exists, deleting it...');
                    await octokit.git.deleteRef({
                      owner: event.repository.owner.login,
                      repo: event.repository.name,
                      ref: `heads/${branchName}`
                    });
                    console.log('Existing branch deleted');
                  }
                } catch (error) {
                  // 404 means branch doesn't exist, which is fine
                  if (error.status !== 404) {
                    throw error;
                  }
                  console.log('Branch does not exist, proceeding with creation');
                }

                // Get main branch ref
                const { data: mainRef } = await octokit.git.getRef({
                  owner: event.repository.owner.login,
                  repo: event.repository.name,
                  ref: 'heads/main'
                });

                // Create new branch
                await octokit.git.createRef({
                  owner: event.repository.owner.login,
                  repo: event.repository.name,
                  ref: `refs/heads/${branchName}`,
                  sha: mainRef.object.sha
                });

                // Get current files from the new branch
                console.log('Getting current file SHAs...');
                const { data: currentReadme } = await octokit.repos.getContent({
                  owner: event.repository.owner.login,
                  repo: event.repository.name,
                  path: 'README.md',
                  ref: branchName
                });

                const { data: currentCsv } = await octokit.repos.getContent({
                  owner: event.repository.owner.login,
                  repo: event.repository.name,
                  path: 'prompts.csv',
                  ref: branchName
                });

                // Update files with correct author
                console.log('Updating README.md...');
                await octokit.repos.createOrUpdateFileContents({
                  owner: event.repository.owner.login,
                  repo: event.repository.name,
                  path: 'README.md',
                  message: promptsArray.length === 1
                    ? `feat: Add "${promptsArray[0].actName}" to README`
                    : `feat: Add multiple prompts to README`,
                  content: Buffer.from(readmeContent).toString('base64'),
                  branch: branchName,
                  sha: currentReadme.sha,
                  committer: {
                    name: pr.user.login,
                    email: `${pr.user.login}@users.noreply.github.com`
                  },
                  author: {
                    name: pr.user.login,
                    email: `${pr.user.login}@users.noreply.github.com`
                  }
                });

                console.log('Updating prompts.csv...');
                await octokit.repos.createOrUpdateFileContents({
                  owner: event.repository.owner.login,
                  repo: event.repository.name,
                  path: 'prompts.csv',
                  message: promptsArray.length === 1
                    ? `feat: Add "${promptsArray[0].actName}" to prompts.csv`
                    : `feat: Add multiple prompts to prompts.csv`,
                  content: Buffer.from(csvContent).toString('base64'),
                  branch: branchName,
                  sha: currentCsv.sha,
                  committer: {
                    name: pr.user.login,
                    email: `${pr.user.login}@users.noreply.github.com`
                  },
                  author: {
                    name: pr.user.login,
                    email: `${pr.user.login}@users.noreply.github.com`
                  }
                });

                // Create new PR
                const prTitle = promptsArray.length === 1
                  ? `feat: Add "${promptsArray[0].actName}"`
                  : `feat: Add multiple prompts (${promptsArray.map(p => `"${p.actName}"`).join(', ')})`;

                const prBody = promptsArray.length === 1
                  ? `This PR supersedes #${issueNumber} with proper formatting. Original PR by @${pr.user.login}. Added "${promptsArray[0].actName}" to README.md and prompts.csv, preserving original attribution.`
                  : `This PR supersedes #${issueNumber} with proper formatting. Original PR by @${pr.user.login}.\n\nAdded the following prompts:\n${promptsArray.map(p => `- "${p.actName}"`).join('\n')}\n\nAll prompts have been added to README.md and prompts.csv, preserving original attribution.`;

                const { data: newPr } = await octokit.pulls.create({
                  owner: event.repository.owner.login,
                  repo: event.repository.name,
                  title: prTitle,
                  head: branchName,
                  base: 'main',
                  body: prBody
                });

                // Comment on original PR
                await octokit.issues.createComment({
                  owner: event.repository.owner.login,
                  repo: event.repository.name,
                  issue_number: issueNumber,
                  body: `I've created a new PR #${newPr.number} with your contribution properly formatted. This PR will be closed in favor of the new one.`
                });

                // Close original PR
                await octokit.pulls.update({
                  owner: event.repository.owner.login,
                  repo: event.repository.name,
                  pull_number: issueNumber,
                  state: 'closed'
                });

                console.log(`Created new PR #${newPr.number} and closed original PR #${issueNumber}`);

              } catch (error) {
                console.error('Error details:', error);
                await octokit.issues.createComment({
                  owner: event.repository.owner.login,
                  repo: event.repository.name,
                  issue_number: issueNumber,
                  body: `❌ Error while trying to create new PR:\n\`\`\`\n${error.message}\n\`\`\``
                });
              }
              return;
            }

            // Handle rename command specifically
            if (aiCommand.startsWith('rename') || aiCommand === 'suggest title') {
              if (!isPullRequest) {
                await octokit.issues.createComment({
                  owner: event.repository.owner.login,
                  repo: event.repository.name,
                  issue_number: issueNumber,
                  body: '❌ The rename command can only be used on pull requests.'
                });
                return;
              }

              // Get PR details for context
              const { data: pr } = await octokit.pulls.get({
                owner: event.repository.owner.login,
                repo: event.repository.name,
                pull_number: issueNumber
              });

              // Get the list of files changed in the PR
              const { data: files } = await octokit.pulls.listFiles({
                owner: event.repository.owner.login,
                repo: event.repository.name,
                pull_number: issueNumber
              });

              // Process file changes
              const fileChanges = await Promise.all(files.map(async file => {
                if (file.status === 'removed') {
                  return `Deleted: ${file.filename}`;
                }

                // Get file content for added or modified files
                if (file.status === 'added' || file.status === 'modified') {
                  const patch = file.patch || '';
                  return `${file.status === 'added' ? 'Added' : 'Modified'}: ${file.filename}\nChanges:\n${patch}`;
                }

                return `${file.status}: ${file.filename}`;
              }));

              const completion = await openai.chat.completions.create({
                model: "gpt-3.5-turbo",
                messages: [
                  {
                    role: "system",
                    content: "You are a helpful assistant that generates clear and concise pull request titles. Follow these rules:\n1. Use conventional commit style (feat:, fix:, docs:, etc.)\n2. Focus on WHAT changed, not HOW or WHERE\n3. Keep it short and meaningful\n4. Don't mention file names or technical implementation details\n5. Return ONLY the new title, nothing else\n\nGood examples:\n- feat: Add \"Act as a Career Coach\"\n- fix: Correct typo in Linux Terminal prompt\n- docs: Update installation instructions\n- refactor: Improve error handling"
                  },
                  {
                    role: "user",
                    content: `Based on these file changes, generate a concise PR title:\n\n${fileChanges.join('\n\n')}`
                  }
                ],
                temperature: 0.5,
                max_tokens: 60
              });

              const newTitle = completion.choices[0].message.content.trim();

              // Update PR title
              await octokit.pulls.update({
                owner: event.repository.owner.login,
                repo: event.repository.name,
                pull_number: issueNumber,
                title: newTitle
              });

              // Add comment about the rename
              await octokit.issues.createComment({
                owner: event.repository.owner.login,
                repo: event.repository.name,
                issue_number: issueNumber,
                body: `✨ Updated PR title to: "${newTitle}"\n\nBased on the following changes:\n\`\`\`diff\n${fileChanges.join('\n')}\n\`\`\``
              });
              return;
            }

            // Handle other commands
            const completion = await openai.chat.completions.create({
              model: "gpt-3.5-turbo",
              messages: [
                {
                  role: "system",
                  content: "You are a helpful AI assistant that helps with GitHub repositories. You can suggest code changes, fix issues, and improve code quality."
                },
                {
                  role: "user",
                  content: aiCommand
                }
              ],
              temperature: 0.7,
              max_tokens: 2000
            });

            const response = completion.choices[0].message.content;

            // If response contains code changes, create a new branch and PR
            if (response.includes('```')) {
              const branchName = `ai-bot/fix-${issueNumber}`;

              // Create new branch
              const defaultBranch = event.repository.default_branch;
              const ref = await octokit.git.getRef({
                owner: event.repository.owner.login,
                repo: event.repository.name,
                ref: `heads/${defaultBranch}`
              });

              await octokit.git.createRef({
                owner: event.repository.owner.login,
                repo: event.repository.name,
                ref: `refs/heads/${branchName}`,
                sha: ref.data.object.sha
              });

              // Extract code changes and file paths from response
              const codeBlocks = response.match(/```[\s\S]*?```/g);
              for (const block of codeBlocks) {
                const [_, filePath, ...codeLines] = block.split('\n');
                const content = Buffer.from(codeLines.join('\n')).toString('base64');

                await octokit.repos.createOrUpdateFileContents({
                  owner: event.repository.owner.login,
                  repo: event.repository.name,
                  path: filePath,
                  message: `AI Bot: Apply suggested changes for #${issueNumber}`,
                  content,
                  branch: branchName
                });
              }

              // Create PR
              await octokit.pulls.create({
                owner: event.repository.owner.login,
                repo: event.repository.name,
                title: `AI Bot: Fix for #${issueNumber}`,
                body: `This PR was automatically generated in response to #${issueNumber}\n\nChanges proposed:\n${response}`,
                head: branchName,
                base: defaultBranch
              });
            }

            // Add comment with response
            await octokit.issues.createComment({
              owner: event.repository.owner.login,
              repo: event.repository.name,
              issue_number: issueNumber,
              body: `AI Bot Response:\n\n${response}`
            });
          }

          main().catch(error => {
            console.error('Error:', error);
            process.exit(1);
          });
          EOF

      - name: Handle errors
        if: failure()
        uses: actions/github-script@v6
        with:
          script: |
            const issueNumber = context.issue.number || context.payload.pull_request.number;
            await github.rest.issues.createComment({
              owner: context.repo.owner,
              repo: context.repo.repo,
              issue_number: issueNumber,
              body: '❌ Sorry, there was an error processing your command. Please try again or contact the repository maintainers.'
            });
