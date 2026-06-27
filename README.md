import json
import hashlib
from pathlib import Path
from datetime import datetime


class FolderSnapshot:

    def __init__(self, folder):
        self.folder = Path(folder)
        self.snapshot_file = self.folder / ".snapshot.json"

    def file_hash(self, path):
        sha = hashlib.sha256()

        with open(path, "rb") as f:
            while chunk := f.read(8192):
                sha.update(chunk)

        return sha.hexdigest()

    def create_snapshot(self):

        snapshot = {}

        for file in self.folder.rglob("*"):

            if not file.is_file():
                continue

            if file.name == ".snapshot.json":
                continue

            relative = str(file.relative_to(self.folder))

            snapshot[relative] = {
                "size": file.stat().st_size,
                "modified": file.stat().st_mtime,
                "hash": self.file_hash(file)
            }

        with open(self.snapshot_file, "w") as f:
            json.dump(snapshot, f, indent=4)

        print("Snapshot created.")
        print(f"Files indexed: {len(snapshot)}")

    def compare(self):

        if not self.snapshot_file.exists():
            print("No snapshot found.")
            return

        with open(self.snapshot_file) as f:
            old = json.load(f)

        current = {}

        for file in self.folder.rglob("*"):

            if not file.is_file():
                continue

            if file.name == ".snapshot.json":
                continue

            relative = str(file.relative_to(self.folder))

            current[relative] = {
                "size": file.stat().st_size,
                "modified": file.stat().st_mtime,
                "hash": self.file_hash(file)
            }

        old_files = set(old.keys())
        current_files = set(current.keys())

        added = current_files - old_files
        removed = old_files - current_files

        changed = []

        for file in old_files & current_files:

            if old[file]["hash"] != current[file]["hash"]:
                changed.append(file)

        print("\n===== REPORT =====")

        print(f"Added   : {len(added)}")
        for f in sorted(added):
            print(" +", f)

        print(f"\nRemoved : {len(removed)}")
        for f in sorted(removed):
            print(" -", f)

        print(f"\nModified: {len(changed)}")
        for f in changed:
            print(" *", f)

        print("\nGenerated:", datetime.now())


def main():

    folder = input("Folder path: ").strip()

    tool = FolderSnapshot(folder)

    print("\n1. Create Snapshot")
    print("2. Compare Folder")

    choice = input("\nChoice: ")

    if choice == "1":
        tool.create_snapshot()

    elif choice == "2":
        tool.compare()

    else:
        print("Invalid choice")


if __name__ == "__main__":
    main()
