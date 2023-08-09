Generated via:

```
#!/usr/bin/env bash

# make-stacked.sh Make a series of stacked branches
# -------------------------------------------------
# Copyright 2023 Tyler Cipriani <tyler@tylercipriani.com>
# License: GPLv3
set -euo pipefail

REMOTE=

if (( $# > 0 )); then
    REMOTE="$@"
fi

# This script is used to archive a Gerrit repository to GitLab.
TMPDIR=$(mktemp -d -t stacked-patches-XXXXXX)

# ------------------ Utility functions ------------------
gitc() {
  git -C "$TMPDIR" "$@"
}

ask() {
    # http://djm.me/ask
    while true; do

        if [ "${2:-}" = "Y" ]; then
            prompt="Y/n"
            default=Y
        elif [ "${2:-}" = "N" ]; then
            prompt="y/N"
            default=N
        else
            prompt="y/n"
            default=
        fi

        # Ask the question - use /dev/tty in case stdin is redirected from somewhere else
        read -p "$1 [$prompt] " REPLY </dev/tty

        # Default?
        if [ -z "$REPLY" ]; then
            REPLY=$default
        fi

        # Check if the reply is valid
        case "$REPLY" in
            Y*|y*) return 0 ;;
            N*|n*) return 1 ;;
        esac

    done
}

# ------------------ The MEAT! ------------------

push_remote() {
    if ask "Push to remote $REMOTE?"; then
        echo "Pushing to remote $REMOTE"
        gitc checkout main
        gitc push --set-upstream origin main
        for i in {1..10}; do
            branch="$(printf "%02d\n" "$i")"
            gitc push --set-upstream origin work/branch/"$branch"
        done
    else
        echo "Skipping push to remote $REMOTE"
    fi
}

create_remote() {
    echo "Creating remote $REMOTE"
    gitc remote add origin "$REMOTE"
    gitc pull origin main || :
}

create_repo() {
    # Create local branch
    gitc init --initial-branch=main
    if [ -z "$REMOTE" ]; then
        echo "No remote specified, skipping remote creation"
    else
        create_remote
    fi
    printf "Generated via:\n\n\`\`\`\n" > "$TMPDIR/README.md"
    cat "$0" >> "$TMPDIR/README.md"
    printf "\n\`\`\`" >> "$TMPDIR/README.md"
    gitc add "$TMPDIR/README.md"
    gitc commit -m "Initial commit"

    for i in {1..10}; do
        branch="$(printf "%02d\n" "$i")"
        gitc checkout -b work/branch/"$branch"
        echo "work/branch/$branch" > "$TMPDIR/${branch}.txt"
        gitc add "$TMPDIR/${branch}.txt"
        gitc commit -m "work/branch/$branch"
    done
    push_remote
    echo "Cherry picking onto main"
    gitc checkout main
    for i in {1..10}; do
        branch="$(printf "%02d\n" "$i")"
        gitc cherry-pick $(gitc rev-parse work/branch/"$branch")
    done
}

main() {
    create_repo
    echo "$TMPDIR"
}

main "$@"

```