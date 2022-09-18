# **L5 Using Git Commit Message Templates**

## Template File

Here is my template*, which i put in a file called `.gitmessage` in my home directory:

```
# Title: Summary, imperative, start upper case, don't end with a period
# No more than 50 chars. #### 50 chars is here:  #

# Remember blank line between title and body.

# Body: Explain *what* and *why* (not *how*). Include task ID (Jira issue).
# Wrap at 72 chars. ################################## which is here:  #


# At the end: Include Co-authored-by for all contributors. 
# Include at least one empty line before it. Format: 
# Co-authored-by: name <user@users.noreply.github.com>
#
# How to Write a Git Commit Message:
# https://chris.beams.io/posts/git-commit/
#
# 1. Separate subject from body with a blank line
# 2. Limit the subject line to 50 characters
# 3. Capitalize the subject line
# 4. Do not end the subject line with a period
# 5. Use the imperative mood in the subject line
# 6. Wrap the body at 72 characters
# 7. Use the body to explain what and why vs. how
```

## Git Configuration


**To tell Git to use the template file (globally, not just in the current repo), I used the following command:**

```
git config --global commit.template ~/.gitmessage
```

### Git push rule

```
^(feature|hotfix)\/(([a-z,A-Z]+))(-)(\d*)(:)([a-z,0–9])
```

```
e.g. feature/JIR-124:test commit message
```

```
(^(I|i)ssue)(-)(\d*)(:)([a-z,0–9])

(^(I|i)ssue)(-)(\d*)(:)\/*
```

**Merge Request Template:** 

```
- **[Project-name(exp: FA|PFM)] - [Issue No.] :** [commit summay]<br />
- **Issue ID:**
- **What and how:**[high level overview of the implementation]<br />
- **Reviewed by:** [name(s)]<br />
- **QA Validate:** [Whether the code is pass the QA test]<br />
- **Related Info:** [Is there any other info related like similar external tickets, commit]<br />
- **Possible Impact:** [does the change have impact on other modules]<br />
- **Testing Suggestion:** [Add if there is]<br />
```
