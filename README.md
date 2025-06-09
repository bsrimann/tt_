# tt_\



@echo off
echo Starting Documentation Time Tracker...
echo.

:: Check if Python is installed
python --version >nul 2>&1
if %ERRORLEVEL% NEQ 0 (
    echo Python is not installed or not in your PATH.
    echo Please install Python from https://www.python.org/downloads/
    echo.
    pause
    exit /b 1
)

:: Check if the script exists
if not exist "tt.py" (
    echo Error: tt.py not found .
    echo Please make sure the script is in the same folder as this batch file.
    echo.
    pause
    exit /b 1
)

:: Run the time tracker

C:\Users\sri\AppData\Local\Programs\Python\Python313\python.exe tt.py


:: If the script exits with an error
if %ERRORLEVEL% NEQ 0 (
    echo.
    echo The time tracker encountered an error.
    pause
)

exit /b 0



***************************************************************************************************8
************************************		tt.py		************************************


import time
import datetime
import csv
import os
import tkinter as tk
from tkinter import messagebox, ttk
import threading
import platform
import subprocess
import ctypes
from pathlib import Path
import time
import winreg

# For multi-monitor support
try:
    # For Windows
    import win32api
    import win32con
    HAS_WIN32 = True
except ImportError:
    HAS_WIN32 = False
    print("Note: win32api not available. Some Windows-specific features will use fallback methods.")

class EnhancedTimeTracker:
    def __init__(self):
        self.is_tracking = False
        self.start_time = None
        self.session_count = 0
        self.total_hours = 0
        self.log_file = "documentation_time_log.csv"
        self.break_interval_minutes = 25
        self.break_thread = None
        self.bw_mode_active = False
        self.bw_mode_duration = 20
        self.break_start_time = None
        
        # Create log file if it doesn't exist
        self.ensure_log_file_exists()
        
        # Set up the GUI
        self.setup_gui()
        
        # Load today's stats
        self.load_today_stats()
        
        # Get information about all monitors
        self.get_monitors_info()
        
        # Store original color settings to restore later
        self.store_original_color_settings()
        
        # Start update loop
        self.update_display()
    
    def ensure_log_file_exists(self):
        """Ensure the CSV log file exists with proper headers"""
        if not os.path.exists(self.log_file):
            with open(self.log_file, 'w', newline='', encoding='utf-8') as file:
                writer = csv.writer(file)
                writer.writerow(["Date", "Start Time", "End Time", "Duration (minutes)", "Notes"])
    
    def setup_gui(self):
        """Set up the GUI interface"""
        self.root = tk.Tk()
        self.root.title("Documentation Time Tracker")
        self.root.geometry("450x500")
        self.root.resizable(True, True)
        
        # Create main frame with padding
        main_frame = ttk.Frame(self.root, padding="10")
        main_frame.grid(row=0, column=0, sticky=(tk.W, tk.E, tk.N, tk.S))
        
        # Configure grid weights for responsive design
        self.root.columnconfigure(0, weight=1)
        self.root.rowconfigure(0, weight=1)
        main_frame.columnconfigure(1, weight=1)
        
        # Status section
        ttk.Label(main_frame, text="Status:", font=('Arial', 10, 'bold')).grid(row=0, column=0, sticky=tk.W, pady=5)
        self.status_var = tk.StringVar(value="Not Tracking")
        status_label = ttk.Label(main_frame, textvariable=self.status_var, font=('Arial', 10))
        status_label.grid(row=0, column=1, sticky=tk.W, pady=5)
        
        # Timer section
        ttk.Label(main_frame, text="Current session:", font=('Arial', 10, 'bold')).grid(row=1, column=0, sticky=tk.W, pady=5)
        self.timer_var = tk.StringVar(value="00:00:00")
        timer_label = ttk.Label(main_frame, textvariable=self.timer_var, font=('Arial', 12, 'bold'))
        timer_label.grid(row=1, column=1, sticky=tk.W, pady=5)
        
        # Session stats
        ttk.Label(main_frame, text="Today's sessions:", font=('Arial', 10, 'bold')).grid(row=2, column=0, sticky=tk.W, pady=5)
        self.session_var = tk.StringVar(value="0")
        ttk.Label(main_frame, textvariable=self.session_var).grid(row=2, column=1, sticky=tk.W, pady=5)
        
        ttk.Label(main_frame, text="Total hours today:", font=('Arial', 10, 'bold')).grid(row=3, column=0, sticky=tk.W, pady=5)
        self.hours_var = tk.StringVar(value="0.0")
        ttk.Label(main_frame, textvariable=self.hours_var).grid(row=3, column=1, sticky=tk.W, pady=5)
        
        # Break settings frame
        break_frame = ttk.LabelFrame(main_frame, text="Break Settings", padding="5")
        break_frame.grid(row=4, column=0, columnspan=2, sticky=(tk.W, tk.E), pady=10)
        break_frame.columnconfigure(1, weight=1)
        
        ttk.Label(break_frame, text="Break every (minutes):").grid(row=0, column=0, sticky=tk.W, pady=2)
        self.break_var = tk.StringVar(value="25")
        break_entry = ttk.Entry(break_frame, textvariable=self.break_var, width=8, validate='key')
        break_entry.grid(row=0, column=1, sticky=tk.W, pady=2)
        break_entry.config(validatecommand=(self.root.register(self.validate_number), '%P'))
        
        ttk.Label(break_frame, text="B&W mode duration (seconds):").grid(row=1, column=0, sticky=tk.W, pady=2)
        self.bw_duration_var = tk.StringVar(value="20")
        bw_entry = ttk.Entry(break_frame, textvariable=self.bw_duration_var, width=8, validate='key')
        bw_entry.grid(row=1, column=1, sticky=tk.W, pady=2)
        bw_entry.config(validatecommand=(self.root.register(self.validate_number), '%P'))
        
        # Auto-end session checkbox
        self.auto_end_session = tk.BooleanVar(value=True)
        ttk.Checkbutton(break_frame, text="End session when taking a break", 
                       variable=self.auto_end_session).grid(row=2, column=0, columnspan=2, sticky=tk.W, pady=5)
        
        # Session notes
        ttk.Label(main_frame, text="Session notes:", font=('Arial', 10, 'bold')).grid(row=5, column=0, sticky=tk.W, pady=5)
        self.notes_var = tk.StringVar()
        notes_entry = ttk.Entry(main_frame, textvariable=self.notes_var, width=40)
        notes_entry.grid(row=5, column=1, sticky=(tk.W, tk.E), pady=5)
        
        # Break status
        ttk.Label(main_frame, text="Next break in:", font=('Arial', 10, 'bold')).grid(row=6, column=0, sticky=tk.W, pady=5)
        self.next_break_var = tk.StringVar(value="--:--")
        ttk.Label(main_frame, textvariable=self.next_break_var, font=('Arial', 10)).grid(row=6, column=1, sticky=tk.W, pady=5)
        
        # B&W mode status
        ttk.Label(main_frame, text="B&W Mode Status:", font=('Arial', 10, 'bold')).grid(row=7, column=0, sticky=tk.W, pady=5)
        self.bw_status_var = tk.StringVar(value="Inactive")
        ttk.Label(main_frame, textvariable=self.bw_status_var).grid(row=7, column=1, sticky=tk.W, pady=5)
        
        # Button frame
        button_frame = ttk.Frame(main_frame)
        button_frame.grid(row=8, column=0, columnspan=2, pady=20)
        button_frame.columnconfigure((0, 1), weight=1)
        
        # Buttons
        self.start_button = ttk.Button(button_frame, text="Start Session", command=self.start_tracking)
        self.start_button.grid(row=0, column=0, padx=5, pady=5, sticky=(tk.W, tk.E))
        
        self.stop_button = ttk.Button(button_frame, text="End Session", command=self.stop_tracking, state=tk.DISABLED)
        self.stop_button.grid(row=0, column=1, padx=5, pady=5, sticky=(tk.W, tk.E))
        
        # Test button (can be removed in production)
        test_button = ttk.Button(button_frame, text="Test B&W Mode", command=self.toggle_bw_mode)
        test_button.grid(row=1, column=0, columnspan=2, padx=5, pady=5, sticky=(tk.W, tk.E))
        
        # View logs button
        logs_button = ttk.Button(button_frame, text="View Today's Log", command=self.show_todays_log)
        logs_button.grid(row=2, column=0, columnspan=2, padx=5, pady=5, sticky=(tk.W, tk.E))
        
        # Handle window closing
        self.root.protocol("WM_DELETE_WINDOW", self.on_closing)
    
    def validate_number(self, value):
        """Validate that input is a positive number"""
        if value == "":
            return True
        try:
            num = int(value)
            return num > 0
        except ValueError:
            return False
    
    def show_todays_log(self):
        """Show today's log entries in a popup window"""
        today = datetime.datetime.now().strftime("%Y-%m-%d")
        
        # Create popup window
        log_window = tk.Toplevel(self.root)
        log_window.title(f"Today's Log - {today}")
        log_window.geometry("600x400")
        
        # Create text widget with scrollbar
        text_frame = ttk.Frame(log_window)
        text_frame.pack(fill=tk.BOTH, expand=True, padx=10, pady=10)
        
        text_widget = tk.Text(text_frame, wrap=tk.WORD)
        scrollbar = ttk.Scrollbar(text_frame, orient=tk.VERTICAL, command=text_widget.yview)
        text_widget.configure(yscrollcommand=scrollbar.set)
        
        text_widget.pack(side=tk.LEFT, fill=tk.BOTH, expand=True)
        scrollbar.pack(side=tk.RIGHT, fill=tk.Y)
        
        # Load and display today's entries
        try:
            with open(self.log_file, 'r', newline='', encoding='utf-8') as file:
                reader = csv.reader(file)
                header = next(reader)
                
                text_widget.insert(tk.END, f"Log entries for {today}\n")
                text_widget.insert(tk.END, "=" * 50 + "\n\n")
                
                entries_found = False
                for row in reader:
                    if len(row) >= 4 and row[0] == today:
                        entries_found = True
                        text_widget.insert(tk.END, f"Session: {row[1]} - {row[2]}\n")
                        text_widget.insert(tk.END, f"Duration: {row[3]} minutes\n")
                        if len(row) > 4 and row[4]:
                            text_widget.insert(tk.END, f"Notes: {row[4]}\n")
                        text_widget.insert(tk.END, "-" * 30 + "\n\n")
                
                if not entries_found:
                    text_widget.insert(tk.END, "No sessions recorded for today.")
                    
        except (FileNotFoundError, IOError) as e:
            text_widget.insert(tk.END, f"Error reading log file: {e}")
        
        text_widget.config(state=tk.DISABLED)
    
    def store_original_color_settings(self):
        """Store original color settings to restore later"""
        self.original_color_settings = None
        if platform.system() == "Windows" and HAS_WIN32:
            try:
                self.original_color_settings = True
            except Exception as e:
                print(f"Note: Could not store original color settings: {e}")
    
    def get_monitors_info(self):
        """Get information about all connected monitors"""
        self.monitors = []
        
        if HAS_WIN32 and platform.system() == "Windows":
            try:
                monitors = win32api.EnumDisplayMonitors()
                for monitor in monitors:
                    info = win32api.GetMonitorInfo(monitor[0])
                    monitor_rect = info['Monitor']
                    
                    left, top, right, bottom = monitor_rect
                    width = right - left
                    height = bottom - top
                    
                    self.monitors.append({
                        'left': left,
                        'top': top,
                        'width': width,
                        'height': height,
                        'center_x': left + width // 2,
                        'center_y': top + height // 2
                    })
                print(f"Detected {len(self.monitors)} monitors using win32api")
            except Exception as e:
                print(f"Win32api monitor detection failed: {e}")
                self._fallback_monitor_detection()
        else:
            self._fallback_monitor_detection()
    
    def _fallback_monitor_detection(self):
        """Fallback method when win32api is not available"""
        width = self.root.winfo_screenwidth()
        height = self.root.winfo_screenheight()
        
        self.monitors.append({
            'left': 0,
            'top': 0,
            'width': width,
            'height': height,
            'center_x': width // 2,
            'center_y': height // 2
        })
        
        print(f"Using fallback monitor detection - primary monitor: {width}x{height}")
    
    def start_tracking(self):
        """Start a new tracking session"""
        if not self.is_tracking:
            self.is_tracking = True
            self.start_time = time.time()
            self.break_start_time = time.time()
            self.status_var.set("Tracking")
            self.start_button.config(state=tk.DISABLED)
            self.stop_button.config(state=tk.NORMAL)
            
            # Validate and set break interval
            try:
                self.break_interval_minutes = max(1, int(self.break_var.get()))
            except ValueError:
                self.break_interval_minutes = 25
                self.break_var.set(str(self.break_interval_minutes))
            
            # Validate and set B&W mode duration
            try:
                self.bw_mode_duration = max(1, int(self.bw_duration_var.get()))
            except ValueError:
                self.bw_mode_duration = 20
                self.bw_duration_var.set(str(self.bw_mode_duration))
            
            # Start break reminder thread
            self.start_break_reminder()
            
            print(f"Started tracking session at {datetime.datetime.now().strftime('%H:%M:%S')}")
    
    def stop_tracking(self):
        """Stop the current tracking session"""
        if self.is_tracking:
            end_time = time.time()
            duration_seconds = end_time - self.start_time
            duration_minutes = duration_seconds / 60
            
            # Format timestamps for log
            start_datetime = datetime.datetime.fromtimestamp(self.start_time)
            end_datetime = datetime.datetime.fromtimestamp(end_time)
            date_str = start_datetime.strftime("%Y-%m-%d")
            start_time_str = start_datetime.strftime("%H:%M:%S")
            end_time_str = end_datetime.strftime("%H:%M:%S")
            
            # Write to log file with error handling
            try:
                with open(self.log_file, 'a', newline='', encoding='utf-8') as file:
                    writer = csv.writer(file)
                    writer.writerow([
                        date_str, 
                        start_time_str, 
                        end_time_str, 
                        f"{duration_minutes:.2f}", 
                        self.notes_var.get()
                    ])
                print(f"Session logged: {duration_minutes:.2f} minutes")
            except IOError as e:
                messagebox.showerror("Error", f"Could not save session to log file: {e}")
            
            # Update session stats
            self.session_count += 1
            self.total_hours += duration_minutes / 60
            self.session_var.set(str(self.session_count))
            self.hours_var.set(f"{self.total_hours:.2f}")
            
            # Reset tracking state
            self.is_tracking = False
            self.start_time = None
            self.timer_var.set("00:00:00")
            self.next_break_var.set("--:--")
            self.status_var.set("Not Tracking")
            self.start_button.config(state=tk.NORMAL)
            self.stop_button.config(state=tk.DISABLED)
            self.notes_var.set("")
            
            # Cancel any pending break reminders
            if self.break_thread is not None:
                self.break_thread.cancel()
            
            # Ensure color mode is restored
            if self.bw_mode_active:
                self.restore_color_mode()
    
    def update_display(self):
        """Update the display with current time and break countdown"""
        if self.is_tracking and self.start_time is not None:
            # Update session timer
            elapsed = time.time() - self.start_time
            hours, remainder = divmod(int(elapsed), 3600)
            minutes, seconds = divmod(remainder, 60)
            self.timer_var.set(f"{hours:02d}:{minutes:02d}:{seconds:02d}")
            
            # Update next break countdown
            if self.break_start_time is not None:
                break_elapsed = time.time() - self.break_start_time
                break_remaining = max(0, (self.break_interval_minutes * 60) - break_elapsed)
                b_minutes, b_seconds = divmod(int(break_remaining), 60)
                self.next_break_var.set(f"{b_minutes:02d}:{b_seconds:02d}")
        
        self.root.after(1000, self.update_display)
    
    def start_break_reminder(self):
        """Start the break reminder timer"""
        if self.is_tracking:
            self.break_start_time = time.time()
            seconds = self.break_interval_minutes * 60
            self.break_thread = threading.Timer(seconds, self.trigger_break)
            self.break_thread.daemon = True
            self.break_thread.start()
    
    def trigger_break(self):
        """Trigger the break by enabling B&W mode"""
        if not self.is_tracking:
            return
        
        print(f"Break triggered at {datetime.datetime.now().strftime('%H:%M:%S')}")
        
        # Set B&W mode
        self.toggle_bw_mode()
        
        # If auto-end session is enabled, end the current session
        if self.auto_end_session.get():
            if not self.notes_var.get():
                self.notes_var.set("Session ended for scheduled break")
            
            # Schedule session end after a short delay to allow B&W mode to activate
            self.root.after(1000, self.stop_tracking)
        else:
            # Otherwise, schedule the next reminder if still tracking
            if self.is_tracking:
                self.start_break_reminder()
    
    def toggle_bw_mode(self):
        """Toggle the black and white display mode"""
        if not self.bw_mode_active:
            # Enable B&W mode
            self.enable_bw_mode()
            
            # Schedule restoration after the specified duration
            restore_thread = threading.Timer(self.bw_mode_duration, self.restore_color_mode)
            restore_thread.daemon = True
            restore_thread.start()
            
            # Update status
            self.bw_status_var.set(f"Active - Restoring in {self.bw_mode_duration}s")
        else:
            # B&W mode is already active, just restore color
            self.restore_color_mode()
    
    def enable_bw_mode(self):
        """Enable black and white display mode system-wide"""
        self.bw_mode_active = True
        os_name = platform.system()
        
        try:
            if os_name == "Windows":
                self.enable_bw_mode_windows()
            elif os_name == "Darwin":  # macOS
                self.enable_bw_mode_macos()
            elif os_name == "Linux":
                self.enable_bw_mode_linux()
            
            self.bw_status_var.set("Active")
        except Exception as e:
            print(f"Error enabling B&W mode: {e}")
            # Fallback to simple notification
            messagebox.showinfo("Break Time!", 
                              f"Time for a {self.bw_mode_duration}-second break!\n\n"
                              f"B&W mode not available on this system.\n"
                              f"Please take your break now.")
            self.bw_mode_active = False
            self.bw_status_var.set("Fallback notification shown")
    
    def restore_color_mode(self):
        """Restore normal color display mode"""
        if not self.bw_mode_active:
            return
            
        os_name = platform.system()
        
        try:
            if os_name == "Windows":
                self.restore_color_mode_windows()
            elif os_name == "Darwin":  # macOS
                self.restore_color_mode_macos()
            elif os_name == "Linux":
                self.restore_color_mode_linux()
            
            self.bw_status_var.set("Inactive")
            self.bw_mode_active = False
        except Exception as e:
            print(f"Error restoring color mode: {e}")
            self.bw_status_var.set("Inactive (may need manual restore)")
            self.bw_mode_active = False
    
    def enable_bw_mode_windows(self):
        """Enable black and white mode on Windows using multiple methods"""
        try:
            # Method 1: Try Windows 10/11 Color Filters
            success = self._try_windows_color_filters(True)
            if success:
                print("Windows B&W mode enabled via Color Filters")
                return
            
            # Method 2: Try High Contrast mode
            success = self._try_high_contrast_mode(True)
            if success:
                print("Windows B&W mode enabled via High Contrast")
                return
                
            # Method 3: Fallback to user notification
            messagebox.showinfo("Break Time!", 
                               f"Time for a {self.bw_mode_duration}-second break!\n\n"
                               "Manual B&W mode:\n"
                               "• Press Win+Ctrl+C to enable Color Filters\n"
                               "• Or press Win+Alt+PrintScreen for High Contrast")
            print("Windows B&W mode: User notification shown")
                
        except Exception as e:
            raise Exception(f"Failed to enable B&W mode on Windows: {str(e)}")
    
    def restore_color_mode_windows(self):
        """Restore color mode on Windows"""
        try:
            # Try to restore Color Filters
            success = self._try_windows_color_filters(False)
            if success:
                print("Windows color mode restored via Color Filters")
                return
            
            # Try to restore High Contrast
            success = self._try_high_contrast_mode(False)
            if success:
                print("Windows color mode restored via High Contrast")
                return
                
            print("Windows color mode: Manual restoration may be needed")
                
        except Exception as e:
            print(f"Error restoring Windows color mode: {e}")

    def _try_windows_color_filters(self, enable):
        """Try to enable/disable Windows Color Filters (Windows 10/11)"""
        try:
            if HAS_WIN32:
                # Check current state first
                current_state = self._is_color_filter_active()
                
                # Only toggle if we need to change state
                if (enable and not current_state) or (not enable and current_state):
                    return self._simulate_color_filter_hotkey()
                else:
                    # Already in desired state
                    return True
            else:
                return self._try_keyboard_shortcut_colorfilters(enable)
                
        except Exception:
            return False

    def _is_color_filter_active(self):
        """Check if color filters are currently active by reading registry"""
        try:
            import winreg
            key_path = r"SOFTWARE\Microsoft\ColorFiltering"
            key = winreg.OpenKey(winreg.HKEY_CURRENT_USER, key_path, 0, winreg.KEY_READ)
            active, _ = winreg.QueryValueEx(key, "Active")
            winreg.CloseKey(key)
            return bool(active)
        except:
            # If we can't read registry, assume it's off
            return False

    def _simulate_color_filter_hotkey(self):
        """Simulate Win+Ctrl+C to toggle Windows Color Filters"""
        try:
            # Virtual-Key Codes
            VK_CONTROL = 0x11
            VK_LWIN = 0x5B
            VK_C = 0x43
            KEYEVENTF_KEYUP = 0x0002
            
            # Press keys down
            ctypes.windll.user32.keybd_event(VK_LWIN, 0, 0, 0)
            time.sleep(0.01)
            ctypes.windll.user32.keybd_event(VK_CONTROL, 0, 0, 0)
            time.sleep(0.01)
            ctypes.windll.user32.keybd_event(VK_C, 0, 0, 0)
            time.sleep(0.05)
            
            # Release keys (in reverse order)
            ctypes.windll.user32.keybd_event(VK_C, 0, KEYEVENTF_KEYUP, 0)
            time.sleep(0.01)
            ctypes.windll.user32.keybd_event(VK_CONTROL, 0, KEYEVENTF_KEYUP, 0)
            time.sleep(0.01)
            ctypes.windll.user32.keybd_event(VK_LWIN, 0, KEYEVENTF_KEYUP, 0)
            
            # Small delay to let Windows process the toggle
            time.sleep(0.1)
            return True
            
        except Exception:
            return False

    # Even simpler version - just always toggle without checking state
    def _try_windows_color_filters_simple(self, enable):
        """Ultra-simple version - just simulate the hotkey"""
        try:
            if HAS_WIN32:
                return self._simulate_color_filter_hotkey()
            else:
                return self._try_keyboard_shortcut_colorfilters(enable)
        except Exception:
            return False

    def _try_keyboard_shortcut_colorfilters(self, enable):
        """Try using keyboard shortcut for color filters"""
        try:
            # Windows 10/11 shortcut: Win+Ctrl+C
            if HAS_WIN32:
                ctypes.windll.user32.keybd_event(0x5B, 0, 0, 0)  # Win key down
                ctypes.windll.user32.keybd_event(0x11, 0, 0, 0)  # Ctrl down
                ctypes.windll.user32.keybd_event(0x43, 0, 0, 0)  # 'C' down
                time.sleep(0.1)
                ctypes.windll.user32.keybd_event(0x43, 0, 2, 0)  # 'C' up
                ctypes.windll.user32.keybd_event(0x11, 0, 2, 0)  # Ctrl up
                ctypes.windll.user32.keybd_event(0x5B, 0, 2, 0)  # Win key up
                return True
            else:
                # Try with pyautogui if available
                try:
                    import pyautogui
                    pyautogui.hotkey('win', 'ctrl', 'c')
                    return True
                except ImportError:
                    return False
        except Exception:
            return False
    
    def _try_high_contrast_mode(self, enable):
        """Try to enable/disable High Contrast mode"""
        try:
            # Windows shortcut: Left Alt + Left Shift + Print Screen
            if HAS_WIN32:
                ctypes.windll.user32.keybd_event(0xA4, 0, 0, 0)  # Left Alt down
                ctypes.windll.user32.keybd_event(0xA0, 0, 0, 0)  # Left Shift down
                ctypes.windll.user32.keybd_event(0x2C, 0, 0, 0)  # Print Screen down
                time.sleep(0.1)
                ctypes.windll.user32.keybd_event(0x2C, 0, 2, 0)  # Print Screen up
                ctypes.windll.user32.keybd_event(0xA0, 0, 2, 0)  # Left Shift up
                ctypes.windll.user32.keybd_event(0xA4, 0, 2, 0)  # Left Alt up
                
                # Wait a moment then press Enter to confirm if dialog appears
                time.sleep(1)
                ctypes.windll.user32.keybd_event(0x0D, 0, 0, 0)  # Enter down
                ctypes.windll.user32.keybd_event(0x0D, 0, 2, 0)  # Enter up
                
                return True
            else:
                try:
                    import pyautogui
                    pyautogui.hotkey('alt', 'shift', 'printscreen')
                    time.sleep(1)
                    pyautogui.press('enter')
                    return True
                except ImportError:
                    return False
        except Exception:
            return False
    
    def enable_bw_mode_macos(self):
        """Enable black and white mode on macOS"""
        try:
            # Use macOS accessibility shortcut - Ctrl+Option+Cmd+8
            subprocess.run(["osascript", "-e", 
                           "tell application \"System Events\" to key code 28 using {command down, option down, control down}"], 
                           check=True)
            print("macOS B&W mode enabled")
        except Exception as e:
            raise Exception(f"Failed to enable B&W mode on macOS: {str(e)}")
    
    def restore_color_mode_macos(self):
        """Restore color mode on macOS"""
        try:
            # Toggle the same command to restore - Ctrl+Option+Cmd+8
            subprocess.run(["osascript", "-e", 
                           "tell application \"System Events\" to key code 28 using {command down, option down, control down}"], 
                           check=True)
            print("macOS color mode restored")
        except Exception as e:
            print(f"Error restoring macOS color mode: {e}")
    
    def enable_bw_mode_linux(self):
        """Enable black and white mode on Linux"""
        try:
            # For GNOME desktop environment
            subprocess.run(["gsettings", "set", "org.gnome.desktop.a11y.magnifier", 
                           "invert-lightness", "true"], check=True)
            print("Linux B&W mode enabled (GNOME)")
        except subprocess.CalledProcessError:
            # Try alternative method for other desktop environments
            try:
                subprocess.run(["xrandr", "--output", 
                               "$(xrandr | grep ' connected' | head -n1 | cut -d' ' -f1)", 
                               "--gamma", "1:1:1", "--brightness", "0.5"], shell=True, check=True)
                print("Linux B&W mode enabled (xrandr fallback)")
            except Exception as e:
                raise Exception(f"Failed to enable B&W mode on Linux: {str(e)}")
        except Exception as e:
            raise Exception(f"Failed to enable B&W mode on Linux: {str(e)}")
    
    def restore_color_mode_linux(self):
        """Restore color mode on Linux"""
        try:
            # For GNOME desktop environment
            subprocess.run(["gsettings", "set", "org.gnome.desktop.a11y.magnifier", 
                           "invert-lightness", "false"], check=True)
            print("Linux color mode restored (GNOME)")
        except subprocess.CalledProcessError:
            # Try alternative restoration
            try:
                subprocess.run(["xrandr", "--output", 
                               "$(xrandr | grep ' connected' | head -n1 | cut -d' ' -f1)", 
                               "--gamma", "1:1:1", "--brightness", "1.0"], shell=True, check=True)
                print("Linux color mode restored (xrandr fallback)")
            except Exception as e:
                print(f"Error restoring Linux color mode: {e}")
        except Exception as e:
            print(f"Error restoring Linux color mode: {e}")
    
    def load_today_stats(self):
        """Load statistics for today's sessions"""
        today = datetime.datetime.now().strftime("%Y-%m-%d")
        self.session_count = 0
        self.total_hours = 0
        
        try:
            with open(self.log_file, 'r', newline='', encoding='utf-8') as file:
                reader = csv.reader(file)
                next(reader)  # Skip header
                
                for row in reader:
                    if len(row) >= 4 and row[0] == today:
                        self.session_count += 1
                        try:
                            duration_minutes = float(row[3])
                            self.total_hours += duration_minutes / 60
                        except (ValueError, IndexError):
                            print(f"Warning: Could not parse duration from row: {row}")
        except (FileNotFoundError, IOError) as e:
            print(f"Note: Could not load today's stats: {e}")
        
        self.session_var.set(str(self.session_count))
        self.hours_var.set(f"{self.total_hours:.2f}")
    
    def on_closing(self):
        """Handle application closing"""
        if self.is_tracking:
            result = messagebox.askyesnocancel(
                "Session Active", 
                "You have an active tracking session.\n\n"
                "Do you want to save it before closing?\n\n"
                "Yes = Save and close\n"
                "No = Close without saving\n"
                "Cancel = Don't close"
            )
            
            if result is True:  # Yes - save and close
                self.stop_tracking()
                self.root.destroy()
            elif result is False:  # No - close without saving
                # Cancel any pending break reminders
                if self.break_thread is not None:
                    self.break_thread.cancel()
                
                # Ensure color mode is restored
                if self.bw_mode_active:
                    self.restore_color_mode()
                
                self.root.destroy()
            # Cancel - do nothing
        else:
            # Ensure color mode is restored
            if self.bw_mode_active:
                self.restore_color_mode()
            
            self.root.destroy()
    
    def run(self):
        """Run the application"""
        try:
            self.root.mainloop()
        except Exception as e:
            print(f"Error running application: {e}")
            # Ensure cleanup
            if self.bw_mode_active:
                self.restore_color_mode()


# Simple time tracker class without B&W mode for users who prefer simplicity
class SimpleTimeTracker:
    def __init__(self):
        self.is_tracking = False
        self.start_time = None
        self.session_count = 0
        self.total_hours = 0
        self.log_file = "simple_time_log.csv"
        self.break_interval_minutes = 25
        self.break_thread = None
        
        # Create log file if it doesn't exist
        if not os.path.exists(self.log_file):
            with open(self.log_file, 'w', newline='', encoding='utf-8') as file:
                writer = csv.writer(file)
                writer.writerow(["Date", "Start Time", "End Time", "Duration (minutes)", "Notes"])
        
        # Set up the GUI
        self.setup_gui()
        self.load_today_stats()
        self.update_display()
    
    def setup_gui(self):
        """Set up a simple GUI interface"""
        self.root = tk.Tk()
        self.root.title("Simple Time Tracker")
        self.root.geometry("400x300")
        
        # Create main frame
        main_frame = ttk.Frame(self.root, padding="15")
        main_frame.grid(row=0, column=0, sticky=(tk.W, tk.E, tk.N, tk.S))
        
        self.root.columnconfigure(0, weight=1)
        self.root.rowconfigure(0, weight=1)
        main_frame.columnconfigure(1, weight=1)
        
        # Status and timer
        ttk.Label(main_frame, text="Status:", font=('Arial', 12, 'bold')).grid(row=0, column=0, sticky=tk.W, pady=10)
        self.status_var = tk.StringVar(value="Not Tracking")
        ttk.Label(main_frame, textvariable=self.status_var, font=('Arial', 12)).grid(row=0, column=1, sticky=tk.W, pady=10)
        
        ttk.Label(main_frame, text="Current session:", font=('Arial', 12, 'bold')).grid(row=1, column=0, sticky=tk.W, pady=5)
        self.timer_var = tk.StringVar(value="00:00:00")
        ttk.Label(main_frame, textvariable=self.timer_var, font=('Arial', 14, 'bold')).grid(row=1, column=1, sticky=tk.W, pady=5)
        
        # Daily stats
        ttk.Label(main_frame, text="Today's sessions:", font=('Arial', 12, 'bold')).grid(row=2, column=0, sticky=tk.W, pady=5)
        self.session_var = tk.StringVar(value="0")
        ttk.Label(main_frame, textvariable=self.session_var, font=('Arial', 12)).grid(row=2, column=1, sticky=tk.W, pady=5)
        
        ttk.Label(main_frame, text="Total hours today:", font=('Arial', 12, 'bold')).grid(row=3, column=0, sticky=tk.W, pady=5)
        self.hours_var = tk.StringVar(value="0.0")
        ttk.Label(main_frame, textvariable=self.hours_var, font=('Arial', 12)).grid(row=3, column=1, sticky=tk.W, pady=5)
        
        # Break reminder setting
        ttk.Label(main_frame, text="Break reminder (min):", font=('Arial', 12, 'bold')).grid(row=4, column=0, sticky=tk.W, pady=5)
        self.break_var = tk.StringVar(value="25")
        ttk.Entry(main_frame, textvariable=self.break_var, width=8).grid(row=4, column=1, sticky=tk.W, pady=5)
        
        # Session notes
        ttk.Label(main_frame, text="Session notes:", font=('Arial', 12, 'bold')).grid(row=5, column=0, sticky=tk.W, pady=5)
        self.notes_var = tk.StringVar()
        ttk.Entry(main_frame, textvariable=self.notes_var, width=30).grid(row=5, column=1, sticky=(tk.W, tk.E), pady=5)
        
        # Buttons
        button_frame = ttk.Frame(main_frame)
        button_frame.grid(row=6, column=0, columnspan=2, pady=20)
        
        self.start_button = ttk.Button(button_frame, text="Start Session", command=self.start_tracking)
        self.start_button.pack(side=tk.LEFT, padx=5)
        
        self.stop_button = ttk.Button(button_frame, text="End Session", command=self.stop_tracking, state=tk.DISABLED)
        self.stop_button.pack(side=tk.LEFT, padx=5)
        
        # Handle window closing
        self.root.protocol("WM_DELETE_WINDOW", self.on_closing)
    
    def start_tracking(self):
        """Start tracking time"""
        self.is_tracking = True
        self.start_time = time.time()
        self.status_var.set("Tracking")
        self.start_button.config(state=tk.DISABLED)
        self.stop_button.config(state=tk.NORMAL)
        
        # Set up break reminder
        try:
            self.break_interval_minutes = max(1, int(self.break_var.get()))
        except ValueError:
            self.break_interval_minutes = 25
            self.break_var.set("25")
        
        # Start break reminder
        self.break_thread = threading.Timer(self.break_interval_minutes * 60, self.show_break_reminder)
        self.break_thread.daemon = True
        self.break_thread.start()
    
    def stop_tracking(self):
        """Stop tracking time"""
        if self.is_tracking:
            end_time = time.time()
            duration_minutes = (end_time - self.start_time) / 60
            
            # Format timestamps
            start_datetime = datetime.datetime.fromtimestamp(self.start_time)
            end_datetime = datetime.datetime.fromtimestamp(end_time)
            date_str = start_datetime.strftime("%Y-%m-%d")
            start_time_str = start_datetime.strftime("%H:%M:%S")
            end_time_str = end_datetime.strftime("%H:%M:%S")
            
            # Save to log
            try:
                with open(self.log_file, 'a', newline='', encoding='utf-8') as file:
                    writer = csv.writer(file)
                    writer.writerow([date_str, start_time_str, end_time_str, 
                                   f"{duration_minutes:.2f}", self.notes_var.get()])
            except IOError as e:
                messagebox.showerror("Error", f"Could not save session: {e}")
            
            # Update stats
            self.session_count += 1
            self.total_hours += duration_minutes / 60
            self.session_var.set(str(self.session_count))
            self.hours_var.set(f"{self.total_hours:.2f}")
            
            # Reset UI
            self.is_tracking = False
            self.start_time = None
            self.timer_var.set("00:00:00")
            self.status_var.set("Not Tracking")
            self.start_button.config(state=tk.NORMAL)
            self.stop_button.config(state=tk.DISABLED)
            self.notes_var.set("")
            
            # Cancel break reminder
            if self.break_thread:
                self.break_thread.cancel()
    
    def show_break_reminder(self):
        """Show a simple break reminder"""
        if self.is_tracking:
            messagebox.showinfo("Break Time!", 
                              f"You've been working for {self.break_interval_minutes} minutes.\n"
                              "Time for a short break!")
            
            # Set up next reminder
            self.break_thread = threading.Timer(self.break_interval_minutes * 60, self.show_break_reminder)
            self.break_thread.daemon = True
            self.break_thread.start()
    
    def update_display(self):
        """Update the timer display"""
        if self.is_tracking and self.start_time:
            elapsed = time.time() - self.start_time
            hours, remainder = divmod(int(elapsed), 3600)
            minutes, seconds = divmod(remainder, 60)
            self.timer_var.set(f"{hours:02d}:{minutes:02d}:{seconds:02d}")
        
        self.root.after(1000, self.update_display)
    
    def load_today_stats(self):
        """Load today's statistics"""
        today = datetime.datetime.now().strftime("%Y-%m-%d")
        self.session_count = 0
        self.total_hours = 0
        
        try:
            with open(self.log_file, 'r', newline='', encoding='utf-8') as file:
                reader = csv.reader(file)
                next(reader)  # Skip header
                
                for row in reader:
                    if len(row) >= 4 and row[0] == today:
                        self.session_count += 1
                        try:
                            duration_minutes = float(row[3])
                            self.total_hours += duration_minutes / 60
                        except (ValueError, IndexError):
                            pass
        except (FileNotFoundError, IOError):
            pass
        
        self.session_var.set(str(self.session_count))
        self.hours_var.set(f"{self.total_hours:.2f}")
    
    def on_closing(self):
        """Handle application closing"""
        if self.is_tracking:
            result = messagebox.askyesnocancel(
                "Session Active", 
                "Save current session before closing?"
            )
            
            if result is True:
                self.stop_tracking()
                self.root.destroy()
            elif result is False:
                if self.break_thread:
                    self.break_thread.cancel()
                self.root.destroy()
        else:
            self.root.destroy()
    
    def run(self):
        """Run the simple tracker"""
        self.root.mainloop()


if __name__ == "__main__":
    # Uncomment the version you want to use:
    
    # Full-featured version with B&W mode
    tracker = EnhancedTimeTracker()
    
    # Simple version without B&W mode
    # tracker = SimpleTimeTracker()
    
    tracker.run()
