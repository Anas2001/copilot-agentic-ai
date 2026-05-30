This project uses Beads (`bd`) for agent workflow and issue tracking.

Before starting work:
- Run `bd prime`
- Check ready work with `bd ready`
- Pick one unblocked task

During work:
- Create beads for discovered tasks
- Add dependencies when one task blocks another
- Store architecture decisions using `bd remember`

After work:
- Update the bead with changed files and decisions
- Close the bead only when tests/checks pass