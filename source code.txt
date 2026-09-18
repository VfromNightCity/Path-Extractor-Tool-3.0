import os
import sys
import json
import subprocess
import glob
import shutil
import tempfile
import concurrent.futures
import time
import threading

# 1. DYNAMIC DIRECTORY DETECTION
if getattr(sys, 'frozen', False):
    BASE_DIR = os.path.dirname(sys.executable)
else:
    BASE_DIR = os.path.dirname(os.path.abspath(__file__))

# 2. RELATIVE PATH DEFINITIONS
MODS_FOLDER = os.path.join(BASE_DIR, "archive", "pc", "mod")
WOLVENKIT_CLI = os.path.join(BASE_DIR, "WolvenKit", "WolvenKit.CLI.exe") 

OUTPUT_JSON = os.path.join(BASE_DIR, "archive_contents_database.json")

# 3. GLOBAL PROGRESS TRACKING
progress_lock = threading.Lock()
progress_state = {'current': 0, 'total': 0}

def get_progress_string():
    """Calculates current progress in a thread-safe way."""
    with progress_lock:
        progress_state['current'] += 1
        return f"[{progress_state['current']}/{progress_state['total']}]"

def check_requirements():
    """Validates the environment and provides clear troubleshooting messages."""
    if not os.path.exists(MODS_FOLDER):
        print("\n❌ ENVIRONMENTAL ERROR: Mods folder not found.")
        print(f"   Looked in: {MODS_FOLDER}")
        print("   Solution: Place this tool in the main Cyberpunk 2077 folder.")
        return False
        
    if not os.path.exists(WOLVENKIT_CLI):
        print("\n❌ ENVIRONMENTAL ERROR: Extraction engine missing.")
        print(f"   Looked in: {WOLVENKIT_CLI}")
        print("   Solution: Make sure the 'WolvenKit' folder containing 'WolvenKit.CLI.exe' was extracted alongside this program.")
        return False
        
    return True

def extract_archive(archive_args):
    """Extracts archives with smart error handling and user-friendly translations."""
    archive_path, mod_name = archive_args
    
    safe_name = mod_name.replace(".archive", "")
    unique_temp = os.path.join(tempfile.gettempdir(), f"cp77_{safe_name}")
    
    if os.path.exists(unique_temp):
        try:
            shutil.rmtree(unique_temp, ignore_errors=True)
        except Exception:
            pass 
            
    os.makedirs(unique_temp, exist_ok=True)
    
    cmd = [
        WOLVENKIT_CLI, "unbundle", 
        "-p", archive_path, 
        "-o", unique_temp
    ]
    
    try:
        subprocess.run(cmd, stdout=subprocess.PIPE, stderr=subprocess.STDOUT, text=True, check=True, encoding='utf-8', errors='replace')
    except subprocess.CalledProcessError as e:
        error_log = e.output.lower() if e.output else ""
        prefix = get_progress_string()
        print(f"\n{prefix} ⚠️ Failed to extract: {mod_name}")
        
        if "hostfxr.dll" in error_log or ".net" in error_log or "framework" in error_log or "runtime" in error_log:
            print("   Reason: Your Windows is missing the '.NET Desktop Runtime' required for WolvenKit to work.")
            print("   Solution: Download and install the latest .NET Runtime from the official Microsoft website.")
        elif "corrupt" in error_log or "invalid" in error_log or "header" in error_log:
            print("   Reason: This .archive file is corrupted or uses an obsolete compression format.")
            print("   Solution: Uninstall this mod, as it may cause game crashes.")
        elif "access denied" in error_log or "in use" in error_log:
            print("   Reason: The file is currently locked by another program.")
            print("   Solution: Close the game or the WolvenKit App if they are running.")
        else:
            print("   Reason: Unexpected WolvenKit error.")
            safe_error_msg = e.output.strip()[:200].replace('\n', ' | ') if e.output else "No details available."
            print(f"   Detail: {safe_error_msg}...")
            
        shutil.rmtree(unique_temp, ignore_errors=True)
        return mod_name, []
        
    extracted_files = []
    for root, dirs, files in os.walk(unique_temp):
        for file in files:
            if file.lower().endswith(".mesh"):
                full_path = os.path.join(root, file)
                rel_path = os.path.relpath(full_path, unique_temp)
                rel_path = rel_path.replace("/", "\\")
                extracted_files.append(rel_path)
                
    shutil.rmtree(unique_temp, ignore_errors=True)
    prefix = get_progress_string()
    print(f"{prefix} [{mod_name}] -> {len(extracted_files)} meshes successfully mapped.")
    return mod_name, sorted(extracted_files)

def process_archives(mode):
    if not check_requirements():
        return

    archives = glob.glob(os.path.join(MODS_FOLDER, "*.archive"))
    
    if not archives:
        print("\nℹ️ No .archive files found in the mods folder.")
        return

    archive_database = {}
    
    if mode == 2:
        if os.path.exists(OUTPUT_JSON):
            try:
                with open(OUTPUT_JSON, 'r', encoding='utf-8') as f:
                    archive_database = json.load(f)
                print(f"\n✅ Database loaded. {len(archive_database)} mods already registered.")
            except (json.JSONDecodeError, PermissionError) as e:
                print(f"\n⚠️ Error reading the old database. Creating a new one from scratch. (Detail: {e})")
        else:
            print("\nℹ️ No previous database found. Starting full scan (Option 1).")

    archives_to_process = []
    for archive in archives:
        mod_name = os.path.basename(archive)
        if mode == 2 and mod_name in archive_database:
            continue
        archives_to_process.append((archive, mod_name))

    if not archives_to_process:
        print("\n✅ Your database is already 100% up to date. No new mods found!")
        return

    print(f"\n🚀 Starting fast mapping of {len(archives_to_process)} mod(s). Please wait...\n")

    # Initialization
    progress_state['current'] = 0
    progress_state['total'] = len(archives_to_process)
    start_time = time.time()

    max_threads = min(8, os.cpu_count() or 4)
    with concurrent.futures.ThreadPoolExecutor(max_workers=max_threads) as executor:
        results = executor.map(extract_archive, archives_to_process)
        
        for mod_name, files_list in results:
            if files_list:
                archive_database[mod_name] = files_list

    # Total time calculation
    total_time = int(time.time() - start_time)
    tm, ts = divmod(total_time, 60)

    if archive_database:
        try:
            with open(OUTPUT_JSON, 'w', encoding='utf-8') as out_f:
                json.dump(archive_database, out_f, indent=4)
            print(f"\n🎉 SUCCESS! Database saved with {len(archive_database)} cataloged mods.")
            print(f"📁 File generated: {OUTPUT_JSON}")
            print(f"⏱️ Total time elapsed: {tm}m {ts}s")
        except PermissionError:
            print("\n❌ FATAL ERROR: Failed to save data.")
            print(f"   Reason: The file '{OUTPUT_JSON}' is currently open in another program.")
            print("   Solution: Close any text editor that might be reading the file and try again.")
    else:
        print("\n⚠️ WARNING: No data was extracted. The process finished without saving any new files.")
        print(f"⏱️ Total time elapsed: {tm}m {ts}s")

def main_menu():
    print("=" * 60)
    print("   CYBERPUNK 2077 - MESH EXTRACTOR (PRO VERSION)")
    print("=" * 60)
    print("1 - Full Mapping (Rebuild database from scratch)")
    print("2 - Incremental Mapping (Scan only new/added mods)")
    print("=" * 60)
    
    while True:
        choice = input("Choose an operation (1 or 2): ").strip()
        if choice in ['1', '2']:
            try:
                process_archives(int(choice))
            except Exception as e:
                print(f"\n❌ AN UNEXPECTED CRITICAL ERROR OCCURRED:")
                print(f"   Detail: {e}")
                print("   Solution: Please report this error to the mod author.")
            finally:
                input("\n[Press ENTER to close this window...]")
                break
        else:
            print("Invalid option. Please type 1 or 2.")

if __name__ == "__main__":
    main_menu()