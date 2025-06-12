```py
import tkinter as tk
from tkinter import ttk, filedialog, messagebox
import pandas as pd
import os
from pathlib import Path
import re
from openpyxl import Workbook
from openpyxl.styles import Font, Alignment, PatternFill, Border, Side
from openpyxl.utils.dataframe import dataframe_to_rows

class ModernExcelDiscrepancyApp:
    def __init__(self, root):
        self.root = root
        self.root.title("Wells Fargo - Excel Discrepancy Analyzer")
        self.root.geometry("900x700")
        self.root.resizable(True, True)
        self.root.configure(bg='#FFFFFF')
        
        # Wells Fargo Color Palette
        self.colors = {
            'primary_red': '#D50000',      # Wells Fargo Red
            'dark_red': '#B71C1C',         # Darker red for hover states
            'gold': '#FFB300',             # Wells Fargo Gold accent
            'dark_gray': '#1A1A1A',        # Text color
            'medium_gray': '#666666',       # Secondary text
            'light_gray': '#F5F5F5',       # Background
            'border_gray': '#E0E0E0',      # Borders
            'white': '#FFFFFF',            # Pure white
            'success_green': '#2E7D32',    # Success states
            'warning_orange': '#F57C00',   # Warning states
            'shadow': '#00000010'          # Subtle shadows
        }
        
        # Variables to store file paths and analysis results
        self.source_file_path = tk.StringVar()
        self.target_file_path = tk.StringVar()
        self.analysis_results = None
        self.source_df = None
        self.target_df = None
        
        # Configure styles
        self.configure_styles()
        
        # Create GUI elements
        self.create_widgets()
        
    def configure_styles(self):
        """Configure modern ttk styles with Wells Fargo theme"""
        style = ttk.Style()
        
        # Configure main theme
        style.theme_use('clam')
        
        # Main frame style
        style.configure('Main.TFrame', 
                       background=self.colors['white'],
                       relief='flat')
        
        # Header style
        style.configure('Header.TFrame',
                       background=self.colors['primary_red'],
                       relief='flat')
        
        # Title label style
        style.configure('Title.TLabel',
                       background=self.colors['primary_red'],
                       foreground=self.colors['white'],
                       font=('Segoe UI', 24, 'bold'),
                       anchor='center')
        
        # Subtitle style
        style.configure('Subtitle.TLabel',
                       background=self.colors['primary_red'],
                       foreground=self.colors['white'],
                       font=('Segoe UI', 11),
                       anchor='center')
        
        # Section header style
        style.configure('Section.TLabel',
                       background=self.colors['white'],
                       foreground=self.colors['dark_gray'],
                       font=('Segoe UI', 12, 'bold'))
        
        # Regular label style
        style.configure('Modern.TLabel',
                       background=self.colors['white'],
                       foreground=self.colors['dark_gray'],
                       font=('Segoe UI', 10))
        
        # Entry style
        style.configure('Modern.TEntry',
                       fieldbackground=self.colors['white'],
                       background=self.colors['white'],
                       foreground=self.colors['dark_gray'],
                       borderwidth=2,
                       relief='solid',
                       bordercolor=self.colors['border_gray'],
                       focuscolor=self.colors['primary_red'],
                       font=('Segoe UI', 10),
                       padding=(10, 8))
        
        # Primary button style
        style.configure('Primary.TButton',
                       background=self.colors['primary_red'],
                       foreground=self.colors['white'],
                       font=('Segoe UI', 11, 'bold'),
                       borderwidth=0,
                       focuscolor='none',
                       padding=(20, 12))
        
        style.map('Primary.TButton',
                 background=[('active', self.colors['dark_red']),
                           ('pressed', self.colors['dark_red'])])
        
        # Secondary button style
        style.configure('Secondary.TButton',
                       background=self.colors['white'],
                       foreground=self.colors['primary_red'],
                       font=('Segoe UI', 10),
                       borderwidth=2,
                       relief='solid',
                       bordercolor=self.colors['primary_red'],
                       focuscolor='none',
                       padding=(15, 8))
        
        style.map('Secondary.TButton',
                 background=[('active', self.colors['light_gray'])])
        
        # Browse button style
        style.configure('Browse.TButton',
                       background=self.colors['gold'],
                       foreground=self.colors['dark_gray'],
                       font=('Segoe UI', 9, 'bold'),
                       borderwidth=0,
                       focuscolor='none',
                       padding=(12, 8))
        
        style.map('Browse.TButton',
                 background=[('active', '#FFA000')])
        
        # Status bar style
        style.configure('Status.TLabel',
                       background=self.colors['light_gray'],
                       foreground=self.colors['medium_gray'],
                       font=('Segoe UI', 9),
                       padding=(10, 6),
                       relief='flat')
        
        # Card frame style
        style.configure('Card.TFrame',
                       background=self.colors['white'],
                       relief='solid',
                       borderwidth=1)
        
    def create_widgets(self):
        # Main container
        main_container = ttk.Frame(self.root, style='Main.TFrame')
        main_container.pack(fill=tk.BOTH, expand=True)
        
        # Header section
        self.create_header(main_container)
        
        # Content area with padding
        content_frame = ttk.Frame(main_container, style='Main.TFrame')
        content_frame.pack(fill=tk.BOTH, expand=True, padx=30, pady=20)
        
        # File selection section
        self.create_file_selection(content_frame)
        
        # Action section
        self.create_action_section(content_frame)
        
        # Results section
        self.create_results_section(content_frame)
        
        # Status bar
        self.create_status_bar(main_container)
        
    def create_header(self, parent):
        """Create the Wells Fargo themed header"""
        header_frame = ttk.Frame(parent, style='Header.TFrame')
        header_frame.pack(fill=tk.X)
        
        # Header content with padding
        header_content = ttk.Frame(header_frame, style='Header.TFrame')
        header_content.pack(fill=tk.X, padx=30, pady=25)
        
        # Title and logo area
        title_frame = ttk.Frame(header_content, style='Header.TFrame')
        title_frame.pack(fill=tk.X)
        
        # Main title
        title = ttk.Label(title_frame, text="Excel Discrepancy Analyzer", 
                         style='Title.TLabel')
        title.pack()
        
        # Subtitle
        subtitle = ttk.Label(title_frame, 
                           text="Professional Data Validation & Analysis Tool", 
                           style='Subtitle.TLabel')
        subtitle.pack(pady=(5, 0))
        
    def create_file_selection(self, parent):
        """Create the file selection section"""
        # File selection card
        file_card = self.create_card(parent, "File Selection")
        
        # Source file
        source_frame = ttk.Frame(file_card, style='Main.TFrame')
        source_frame.pack(fill=tk.X, pady=(0, 15))
        
        source_label = ttk.Label(source_frame, text="Source Excel File:", 
                               style='Modern.TLabel')
        source_label.pack(anchor=tk.W, pady=(0, 5))
        
        source_input_frame = ttk.Frame(source_frame, style='Main.TFrame')
        source_input_frame.pack(fill=tk.X)
        source_input_frame.columnconfigure(0, weight=1)
        
        source_entry = ttk.Entry(source_input_frame, textvariable=self.source_file_path, 
                               style='Modern.TEntry')
        source_entry.grid(row=0, column=0, sticky=(tk.W, tk.E), padx=(0, 10))
        
        source_browse = ttk.Button(source_input_frame, text="Browse", 
                                 command=self.browse_source_file, 
                                 style='Browse.TButton')
        source_browse.grid(row=0, column=1)
        
        # Target file
        target_frame = ttk.Frame(file_card, style='Main.TFrame')
        target_frame.pack(fill=tk.X)
        
        target_label = ttk.Label(target_frame, text="Target Excel File:", 
                               style='Modern.TLabel')
        target_label.pack(anchor=tk.W, pady=(0, 5))
        
        target_input_frame = ttk.Frame(target_frame, style='Main.TFrame')
        target_input_frame.pack(fill=tk.X)
        target_input_frame.columnconfigure(0, weight=1)
        
        target_entry = ttk.Entry(target_input_frame, textvariable=self.target_file_path, 
                               style='Modern.TEntry')
        target_entry.grid(row=0, column=0, sticky=(tk.W, tk.E), padx=(0, 10))
        
        target_browse = ttk.Button(target_input_frame, text="Browse", 
                                 command=self.browse_target_file, 
                                 style='Browse.TButton')
        target_browse.grid(row=0, column=1)
        
    def create_action_section(self, parent):
        """Create the action buttons section"""
        action_frame = ttk.Frame(parent, style='Main.TFrame')
        action_frame.pack(fill=tk.X, pady=20)
        
        # Center the buttons
        button_frame = ttk.Frame(action_frame, style='Main.TFrame')
        button_frame.pack(anchor=tk.CENTER)
        
        # Main action button
        analyze_btn = ttk.Button(button_frame, text="🔍 Analyze Discrepancies", 
                               command=self.find_discrepancies, 
                               style='Primary.TButton')
        analyze_btn.pack(side=tk.LEFT, padx=(0, 15))
        
        # Export button
        self.export_btn = ttk.Button(button_frame, text="📊 Export Results", 
                                   command=self.export_results, 
                                   style='Secondary.TButton')
        self.export_btn.pack(side=tk.LEFT)
        self.export_btn.configure(state='disabled')
        
    def create_results_section(self, parent):
        """Create the results display section"""
        results_card = self.create_card(parent, "Analysis Results")
        
        # Text widget with custom styling
        text_frame = ttk.Frame(results_card, style='Main.TFrame')
        text_frame.pack(fill=tk.BOTH, expand=True)
        text_frame.columnconfigure(0, weight=1)
        text_frame.rowconfigure(0, weight=1)
        
        # Create text widget with modern styling
        self.results_text = tk.Text(text_frame, 
                                  wrap=tk.WORD, 
                                  font=('Consolas', 10),
                                  bg=self.colors['white'],
                                  fg=self.colors['dark_gray'],
                                  selectbackground=self.colors['primary_red'],
                                  selectforeground=self.colors['white'],
                                  borderwidth=0,
                                  relief='flat',
                                  padx=15,
                                  pady=15)
        
        # Custom scrollbar
        scrollbar = ttk.Scrollbar(text_frame, orient=tk.VERTICAL, 
                                command=self.results_text.yview)
        self.results_text.configure(yscrollcommand=scrollbar.set)
        
        self.results_text.grid(row=0, column=0, sticky=(tk.W, tk.E, tk.N, tk.S))
        scrollbar.grid(row=0, column=1, sticky=(tk.N, tk.S))
        
        # Configure text tags for colored output
        self.configure_text_tags()
        
        # Placeholder text
        self.show_placeholder_text()
        
    def create_card(self, parent, title):
        """Create a card-style container with title"""
        card_container = ttk.Frame(parent, style='Main.TFrame')
        card_container.pack(fill=tk.BOTH, expand=True, pady=(0, 20))
        
        # Card title
        title_label = ttk.Label(card_container, text=title, style='Section.TLabel')
        title_label.pack(anchor=tk.W, pady=(0, 10))
        
        # Card content with border and shadow effect
        card_frame = ttk.Frame(card_container, style='Card.TFrame')
        card_frame.pack(fill=tk.BOTH, expand=True, padx=2, pady=2)
        
        # Inner content with padding
        content_frame = ttk.Frame(card_frame, style='Main.TFrame')
        content_frame.pack(fill=tk.BOTH, expand=True, padx=20, pady=20)
        
        return content_frame
        
    def create_status_bar(self, parent):
        """Create a modern status bar"""
        status_frame = ttk.Frame(parent, style='Status.TLabel')
        status_frame.pack(fill=tk.X, side=tk.BOTTOM)
        
        self.status_var = tk.StringVar()
        self.status_var.set("Ready for analysis")
        
        status_label = ttk.Label(status_frame, textvariable=self.status_var, 
                               style='Status.TLabel')
        status_label.pack(side=tk.LEFT)
        
        # Version info
        version_label = ttk.Label(status_frame, text="v2.1", 
                                style='Status.TLabel')
        version_label.pack(side=tk.RIGHT)
        
    def configure_text_tags(self):
        """Configure text tags for colored and styled output"""
        self.results_text.tag_configure("header", 
                                       foreground=self.colors['primary_red'],
                                       font=('Segoe UI', 16, 'bold'))
        self.results_text.tag_configure("success", 
                                       foreground=self.colors['success_green'],
                                       font=('Segoe UI', 14, 'bold'))
        self.results_text.tag_configure("normal", 
                                       foreground=self.colors['dark_gray'],
                                       font=('Segoe UI', 12))
        
    def show_placeholder_text(self):
        """Show placeholder text in results area"""
        placeholder = """Welcome to the Excel Discrepancy Analyzer

📋 Instructions:
1. Select your Source Excel file (format: {APPID}_Source_Data.xlsx)
2. Select your Target Excel file (format: {APPID}_Target_Data.xlsx)
3. Click 'Analyze Discrepancies' to get missing entries count
4. Export results in Excel format with VLOOKUP formulas

Ready to analyze your data!"""
        
        self.results_text.insert(tk.END, placeholder, "normal")
        self.results_text.configure(state='disabled')
        
    def browse_source_file(self):
        filename = filedialog.askopenfilename(
            title="Select Source Excel File",
            filetypes=[("Excel files", "*.xlsx *.xls"), ("All files", "*.*")]
        )
        if filename:
            self.source_file_path.set(filename)
            self.validate_filename(filename, "source")
            self.update_status("Source file selected")
    
    def browse_target_file(self):
        filename = filedialog.askopenfilename(
            title="Select Target Excel File",
            filetypes=[("Excel files", "*.xlsx *.xls"), ("All files", "*.*")]
        )
        if filename:
            self.target_file_path.set(filename)
            self.validate_filename(filename, "target")
            self.update_status("Target file selected")
    
    def validate_filename(self, filepath, file_type):
        filename = Path(filepath).name
        if file_type == "source":
            pattern = r'^(.+)_Source_Data\.xlsx?$'
            expected_format = "{APPID}_Source_Data.xlsx"
        else:
            pattern = r'^(.+)_Target_Data\.xlsx?$'
            expected_format = "{APPID}_Target_Data.xlsx"
        
        if not re.match(pattern, filename):
            messagebox.showwarning(
                "Invalid Filename", 
                f"Filename should be in format: {expected_format}\n\n"
                f"Current filename: {filename}",
                icon='warning'
            )
            return False
        return True
    
    def update_status(self, message):
        """Update status bar with message"""
        self.status_var.set(message)
        self.root.update_idletasks()
    
    def find_discrepancies(self):
        if not self.source_file_path.get() or not self.target_file_path.get():
            messagebox.showerror("Missing Files", 
                               "Please select both source and target files before analyzing.",
                               icon='error')
            return
        
        try:
            self.update_status("Loading files...")
            
            # Clear previous results
            self.results_text.configure(state='normal')
            self.results_text.delete(1.0, tk.END)
            
            # Read Excel files
            self.update_status("Reading source file...")
            self.source_df = pd.read_excel(self.source_file_path.get())
            
            self.update_status("Reading target file...")
            self.target_df = pd.read_excel(self.target_file_path.get())
            
            # Validate columns
            if not self.validate_columns(self.source_df, self.target_df):
                return
            
            self.update_status("Normalizing data...")
            # Normalize column names and data
            self.source_df = self.normalize_dataframe(self.source_df, is_source=True)
            self.target_df = self.normalize_dataframe(self.target_df, is_source=False)
            
            self.update_status("Analyzing discrepancies...")
            # Find discrepancies - only missing in target
            missing_in_target = self.find_missing_in_target(self.source_df, self.target_df)
            
            # Store results
            self.analysis_results = missing_in_target
            
            # Display simple results
            self.display_simple_results(missing_in_target)
            
            # Enable export button
            self.export_btn.configure(state='normal')
            
            self.update_status("Analysis completed successfully")
            
        except Exception as e:
            messagebox.showerror("Analysis Error", 
                               f"An error occurred during analysis:\n\n{str(e)}\n\n"
                               "Please check your files and try again.",
                               icon='error')
            self.update_status("Error occurred during analysis")
    
    def validate_columns(self, source_df, target_df):
        # Check source columns
        source_cols = source_df.columns.tolist()
        expected_source = ['USERNAME', 'ENTITLEMENT']
        
        if not all(col.upper() in [c.upper() for c in source_cols] for col in expected_source):
            messagebox.showerror("Invalid Source File", 
                f"Source file must have columns: {expected_source}\n\n"
                f"Found columns: {source_cols}",
                icon='error')
            return False
        
        # Check target columns - flexible column names
        target_cols = target_df.columns.tolist()
        target_cols_upper = [col.upper() for col in target_cols]
        
        # Look for UID or ItemName in first column
        if not any(name in target_cols_upper[0] for name in ['UID', 'ITEMNAME']):
            messagebox.showwarning("Target File Warning", 
                f"Target file first column should contain 'UID' or 'ItemName'\n\n"
                f"Found: {target_cols[0]}",
                icon='warning')
        
        return True
    
    def normalize_dataframe(self, df, is_source=True):
        # Create a copy to avoid modifying original
        df_normalized = df.copy()
        
        # Rename columns to standard names
        if is_source:
            # Find USERNAME and ENTITLEMENT columns (case insensitive)
            col_mapping = {}
            for col in df_normalized.columns:
                if 'USERNAME' in col.upper():
                    col_mapping[col] = 'USERNAME'
                elif 'ENTITLEMENT' in col.upper():
                    col_mapping[col] = 'ENTITLEMENT'
            df_normalized.rename(columns=col_mapping, inplace=True)
        else:
            # For target, use first two columns and rename them
            cols = df_normalized.columns.tolist()
            if len(cols) >= 2:
                df_normalized.rename(columns={
                    cols[0]: 'UID',
                    cols[1]: 'ENTITLEMENT'
                }, inplace=True)
        
        # Clean data: remove leading/trailing spaces, handle NaN
        for col in df_normalized.columns:
            if df_normalized[col].dtype == 'object':
                df_normalized[col] = df_normalized[col].astype(str).str.strip()
                df_normalized[col] = df_normalized[col].replace('nan', '')
        
        # Remove empty rows
        df_normalized = df_normalized.dropna(how='all')
        
        return df_normalized
    
    def find_missing_in_target(self, source_df, target_df):
        """Find entries that are in source but missing in target"""
        # Create sets for comparison
        source_set = set(zip(source_df['USERNAME'], source_df['ENTITLEMENT']))
        target_set = set(zip(target_df['UID'], target_df['ENTITLEMENT']))
        
        # Find missing entries
        missing_in_target = list(source_set - target_set)
        
        return missing_in_target
    
    def display_simple_results(self, missing_in_target):
        """Display simple count of missing entries"""
        self.results_text.configure(state='normal')
        
        # Header
        self.insert_with_tag("DISCREPANCY ANALYSIS RESULTS\n", "header")
        self.insert_with_tag("=" * 50 + "\n\n", "header")
        
        # Show count
        count = len(missing_in_target)
        self.insert_with_tag(f"Missing in Target: {count} entries\n\n", "success")
        
        if count > 0:
            self.insert_with_tag("Use 'Export Results' to generate detailed Excel report with VLOOKUP formulas.\n", "normal")
        else:
            self.insert_with_tag("No discrepancies found! All source entries exist in target.\n", "normal")
        
        self.results_text.configure(state='disabled')
        
    def insert_with_tag(self, text, tag):
        """Insert text with specified tag"""
        self.results_text.insert(tk.END, text, tag)
    
    def export_results(self):
        if self.analysis_results is None or self.source_df is None or self.target_df is None:
            messagebox.showwarning("No Results", 
                                 "No results to export. Please run analysis first.",
                                 icon='warning')
            return
        
        filename = filedialog.asksaveasfilename(
            title="Export Analysis Results",
            defaultextension=".xlsx",
            filetypes=[
                ("Excel files", "*.xlsx"), 
                ("All files", "*.*")
            ]
        )
        
        if filename:
            try:
                self.create_excel_report(filename)
                messagebox.showinfo("Export Successful", 
                                   f"Results exported successfully to:\n\n{filename}\n\n"
                                   "The file contains the complete analysis with VLOOKUP formulas.",
                                   icon='info')
                self.update_status(f"Results exported to {Path(filename).name}")
            except Exception as e:
                messagebox.showerror("Export Error", 
                                   f"Failed to export results:\n\n{str(e)}",
                                   icon='error')
                self.update_status("Export failed")
    
    def create_excel_report(self, filename):
        """Create Excel report matching the format shown in the image"""
        wb = Workbook()
        ws = wb.active
        ws.title = "User Account Testing Form"
        
        # Header styling
        header_font = Font(bold=True, size=12)
        center_alignment = Alignment(horizontal='center', vertical='center')
        header_fill = PatternFill(start_color='D3D3D3', end_color='D3D3D3', fill_type='solid')
        border = Border(
            left=Side(style='thin'),
            right=Side(style='thin'),
            top=Side(style='thin'),
            bottom=Side(style='thin')
        )
        
        # Set up headers
        ws['A1'] = '1CCP'
        ws['A2'] = 'User Account Testing Form'
        ws['A3'] = 'Source Data'
        ws['C3'] = 'Testing'
        ws['F3'] = 'Target Data'
        
        # Column headers
        ws['A4'] = 'User Account ID'
        ws['B4'] = 'Entitlement'
        ws['C4'] = 'Vlookup'
        ws['D4'] = 'Issues?'
        ws['E4'] = 'User ID'
        ws['F4'] = 'Entitlement'
        ws['G4'] = 'ID AND Entitlement'
        
        # Apply header styling
        for cell in ['A1', 'A2', 'A3', 'C3', 'F3']:
            ws[cell].font = header_font
            ws[cell].alignment = center_alignment
            ws[cell].fill = header_fill
        
        for col in ['A4', 'B4', 'C4', 'D4', 'E4', 'F4', 'G4']:
            ws[col].font = header_font
            ws[col].alignment = center_alignment
            ws[col].border = border
        
        # Add source data
        row = 5
        for _, source_row in self.source_df.iterrows():
            ws[f'A{row}'] = source_row['USERNAME']
            ws[f'B{row}'] = source_row['ENTITLEMENT']
            
            # VLOOKUP formula - exact match from image
            vlookup_formula = f'=IFERROR(VLOOKUP("*"&A{row}&"*"&B{row}&"*",I:I,1,0),"#N/A")'
            ws[f'C{row}'] = vlookup_formula
            
            # Issues column
            ws[f'D{row}'] = f'=IF(C{row}="#N/A","Yes","No Issues")'
            
            row += 1
        
        # Add target data starting from column E
        target_row = 5
        for _, tgt_row in self.target_df.iterrows():
            ws[f'E{target_row}'] = tgt_row['UID']
            ws[f'F{target_row}'] = tgt_row['ENTITLEMENT']
            ws[f'G{target_row}'] = f"{tgt_row['UID']} AND {tgt_row['ENTITLEMENT']}"
            target_row += 1
        
        # Create lookup column in column I (for VLOOKUP reference)
        lookup_row = 5
        for _, tgt_row in self.target_df.iterrows():
            # Format: UID-ENTITLEMENT for lookup
            lookup_value = f"{tgt_row['UID']}-{tgt_row['ENTITLEMENT']}"
            ws[f'I{lookup_row}'] = lookup_value
            lookup_row += 1
        
        # Auto-adjust column widths
        for column in ws.columns:
            max_length = 0
            column_letter = column[0].column_letter
            for cell in column:
                try:
                    if len(str(cell.value)) > max_length:
                        max_length = len(str(cell.value))
                except:
                    pass
            adjusted_width = min(max_length + 2, 50)
            ws.column_dimensions[column_letter].width = adjusted_width
        
        # Save the workbook
        wb.save(filename)

def main():
    """Main function to run the application"""
    root = tk.Tk()
    
    # Set minimum window size
    root.minsize(800, 600)
    
    # Center window on screen
    window_width = 900
    window_height = 700
    screen_width = root.winfo_screenwidth()
    screen_height = root.winfo_screenheight()
    
    center_x = int(screen_width/2 - window_width/2)
    center_y = int(screen_height/2 - window_height/2)
    
    root.geometry(f'{window_width}x{window_height}+{center_x}+{center_y}')
    
    # Create and run the application
    app = ModernExcelDiscrepancyApp(root)
    
    # Handle window closing
    def on_closing():
        if messagebox.askokcancel("Quit", "Do you want to quit the Excel Discrepancy Analyzer?"):
            root.destroy()
    
    root.protocol("WM_DELETE_WINDOW", on_closing)
    
    # Start the application
    root.mainloop()

if __name__ == "__main__":
    main()
```
