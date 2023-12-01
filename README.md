import tkinter as tk
from tkinter import messagebox
import peewee as pw
import bcrypt

DATABASE_NAME = "agri_loans.db"
db = pw.SqliteDatabase(DATABASE_NAME)

class BaseModel(pw.Model):
    class Meta:
        database = db

class User(BaseModel):
    username = pw.CharField(unique=True)
    password = pw.CharField()
    role = pw.CharField(default="user")
    employment_status = pw.CharField()
    annual_income = pw.FloatField()
    outstanding_debts = pw.FloatField()
    credit_history = pw.CharField()  # Simple representation (e.g., 'Good', 'Average', 'Poor')

class Loan(BaseModel):
    user = pw.ForeignKeyField(User, backref='loans')
    amount = pw.FloatField()
    status = pw.CharField(default="Pending")
    repayment = pw.FloatField(default=0)

def setup_database():
    db.connect()
    db.create_tables([User, Loan])

def hash_password(password):
    return bcrypt.hashpw(password.encode('utf-8'), bcrypt.gensalt()).decode('utf-8')

def verify_password(stored_password, provided_password):
    return bcrypt.checkpw(provided_password.encode('utf-8'), stored_password.encode('utf-8'))

class App:
    def __init__(self, root):
        print("Initializing the app...")
        self.root = root
        self.root.title("Agri Loans Management System")
        self.current_user = None

        self.main_frame = tk.Frame(self.root)
        self.main_frame.pack()

        self.create_home_buttons()

    def create_home_buttons(self):
        self.clear_frame()

        tk.Button(self.main_frame, text="Register", command=self.register_gui).pack()
        tk.Button(self.main_frame, text="Login", command=self.login_gui).pack()
        tk.Button(self.main_frame, text="Admin Actions", command=self.admin_actions).pack()
        tk.Button(self.main_frame, text="Exit", command=self.root.quit).pack()

    def clear_frame(self):
        for widget in self.main_frame.winfo_children():
            widget.destroy()

    def delete_loan(self, loan):
        try:
            loan.delete_instance()
            messagebox.showinfo("Success", f"Loan ID {loan.id} deleted")
            self.refresh_loans_display()  # Refresh the display
        except Exception as e:
            messagebox.showerror("Error", f"Failed to delete loan: {e}")

    def logout(self):
        self.current_user = None
        messagebox.showinfo("Logged out", "You have successfully logged out.")
        self.create_home_buttons()        

    def admin_dashboard(self):
        self.clear_frame()
        tk.Button(self.main_frame, text="View All Loans", command=self.view_all_loans_gui).pack()
        tk.Button(self.main_frame, text="Approve/Reject Loans", command=self.approve_reject_loans_gui).pack()
        tk.Button(self.main_frame, text="Logout", command=self.logout).pack()

    def view_all_loans_gui(self):
        self.clear_frame()
        loans = Loan.select()
        for loan in loans:
            loan_info = f"Loan ID: {loan.id}, User: {loan.user.username}, Amount: {loan.amount}, Status: {loan.status}, Repaid: {loan.repayment}"
            tk.Label(self.main_frame, text=loan_info).pack()
        tk.Button(self.main_frame, text="Back", command=self.admin_dashboard).pack()

    def approve_reject_loans_gui(self):
        self.clear_frame()
        loans = Loan.select().where(Loan.status == "Pending").join(User)

        for loan in loans:
            user = loan.user
            user_info = (f"User: {user.username}, Employment Status: {user.employment_status}, "
                         f"Annual Income: {user.annual_income}, Outstanding Debts: {user.outstanding_debts}, "
                         f"Credit History: {user.credit_history}")
            loan_info = f"Loan ID: {loan.id}, Amount: {loan.amount}, Status: {loan.status}"
            combined_info = f"{user_info}\n{loan_info}"

            tk.Label(self.main_frame, text=combined_info).pack()

            # Automated decision based on predefined criteria
            if self.should_approve_loan(user):
                tk.Button(self.main_frame, text="Approve", command=lambda l=loan: self.approve_loan(l)).pack()
            else:
                tk.Button(self.main_frame, text="Reject", command=lambda l=loan: self.reject_loan(l)).pack()

        tk.Button(self.main_frame, text="Back", command=self.admin_dashboard).pack()

    def should_approve_loan(self, user):
        # Define your criteria here
        income_threshold = 75000  # Example threshold
        debt_to_income_ratio = user.outstanding_debts / user.annual_income
        return (user.annual_income > income_threshold and 
            user.credit_history == "Good" and 
            user.employment_status == "Employed" and 
            debt_to_income_ratio < 0.4)

    def approve_loan(self, loan):
        loan.status = "Accepted"
        loan.save()
        messagebox.showinfo("Success", f"Loan ID {loan.id} approved")
        self.approve_reject_loans_gui()

    def reject_loan(self, loan):
        loan.status = "Rejected"
        loan.save()
        messagebox.showinfo("Success", f"Loan ID {loan.id} rejected")
        self.approve_reject_loans_gui()

    def refresh_loans_display(self, status_filter=None):
        self.clear_frame()
        query = Loan.select().where(Loan.user == self.current_user)
        if status_filter:
            query = query.where(Loan.status == status_filter)
        for loan in query:
            loan_info = f"Loan ID: {loan.id}, Amount: {loan.amount}, Status: {loan.status}, Repaid: {loan.repayment}"
            loan_label = tk.Label(self.main_frame, text=loan_info)
            loan_label.pack()
            delete_button = tk.Button(self.main_frame, text="Delete", command=lambda l=loan: self.delete_loan(l))
            delete_button.pack()
        tk.Button(self.main_frame, text="Back", command=self.user_dashboard).pack()
        
    def register_gui(self):
        self.clear_frame()

        tk.Label(self.main_frame, text="Username").pack()
        username_entry = tk.Entry(self.main_frame)
        username_entry.pack()
        
        tk.Label(self.main_frame, text="Password").pack()
        password_entry = tk.Entry(self.main_frame, show="*")
        password_entry.pack()

        tk.Label(self.main_frame, text="Employment Status").pack()
        employment_status_entry = tk.Entry(self.main_frame)
        employment_status_entry.pack()

        tk.Label(self.main_frame, text="Annual Income").pack()
        annual_income_entry = tk.Entry(self.main_frame)
        annual_income_entry.pack()

        tk.Label(self.main_frame, text="Outstanding Debts").pack()
        outstanding_debts_entry = tk.Entry(self.main_frame)
        outstanding_debts_entry.pack()

        tk.Label(self.main_frame, text="Credit History").pack()
        credit_history_entry = tk.Entry(self.main_frame)
        credit_history_entry.pack()

        def on_register():
            username = username_entry.get()
            password = password_entry.get()
            employment_status = employment_status_entry.get()
            annual_income = float(annual_income_entry.get())
            outstanding_debts = float(outstanding_debts_entry.get())
            credit_history = credit_history_entry.get()
            hashed_password = hash_password(password)
            try:
                User.create(username=username, password=hashed_password, employment_status=employment_status,
                            annual_income=annual_income, outstanding_debts=outstanding_debts, credit_history=credit_history)
                messagebox.showinfo("Success", "Registration successful!")
                self.create_home_buttons()
            except pw.IntegrityError:
                messagebox.showerror("Error", "Username already exists!")

        tk.Button(self.main_frame, text="Register", command=on_register).pack()

    def login_gui(self):
        self.clear_frame()

        tk.Label(self.main_frame, text="Username").pack()
        username_entry = tk.Entry(self.main_frame)
        username_entry.pack()

        tk.Label(self.main_frame, text="Password").pack()
        password_entry = tk.Entry(self.main_frame, show="*")
        password_entry.pack()

        def on_login():
            username = username_entry.get()
            password = password_entry.get()
            user = User.get_or_none(User.username == username)
            if user and verify_password(user.password, password):
                self.current_user = user
                self.user_dashboard()
            else:
                messagebox.showerror("Error", "Invalid username or password!")

        tk.Button(self.main_frame, text="Login", command=on_login).pack()

    def admin_actions(self):
        self.clear_frame()

        tk.Label(self.main_frame, text="Admin Username").pack()
        admin_username_entry = tk.Entry(self.main_frame)
        admin_username_entry.pack()

        tk.Label(self.main_frame, text="Admin Password").pack()
        admin_password_entry = tk.Entry(self.main_frame, show="*")
        admin_password_entry.pack()

        def on_admin_login():
            username = admin_username_entry.get()
            password = admin_password_entry.get()
        # Simple check for admin credentials
            if username == "admin" and password == "adminpass":
                self.admin_dashboard()
            else:
                messagebox.showerror("Error", "Invalid admin credentials")

        tk.Button(self.main_frame, text="Login", command=on_admin_login).pack()

    def apply_loan_gui(self):
        self.clear_frame()

        tk.Label(self.main_frame, text="Loan Amount").pack()
        loan_amount_entry = tk.Entry(self.main_frame)
        loan_amount_entry.pack()

        def on_apply_loan():
            loan_amount = float(loan_amount_entry.get())
            try:
                Loan.create(user=self.current_user, amount=loan_amount, status="Pending")
                messagebox.showinfo("Success", "Loan application submitted!")
            except Exception as e:
                messagebox.showerror("Error", f"Failed to submit loan application: {e}")
            
        tk.Button(self.main_frame, text="Submit", command=on_apply_loan).pack()
        tk.Button(self.main_frame, text="Back", command=self.user_dashboard).pack()
    def create_submit_and_back_buttons(self):
        tk.Button(self.main_frame, text="Submit", command=self.submit_action).pack()
        
        tk.Button(self.main_frame, text="Back", command=self.go_back).pack()

    def submit_action(self):
        print("Submit button clicked")

    def go_back(self):
        self.clear_frame()
        self.create_home_buttons()
    def user_dashboard(self):
        self.clear_frame()

        tk.Button(self.main_frame, text="Apply for Loan", command=self.apply_loan_gui).pack()
        tk.Button(self.main_frame, text="View My Loans", command=self.view_loans_gui).pack()
        tk.Button(self.main_frame, text="Repay a Loan", command=self.repay_loan_gui).pack()
        tk.Button(self.main_frame, text="Logout", command=self.create_home_buttons).pack()
  
    def view_loans_gui(self):
        self.clear_frame()
        tk.Button(self.main_frame, text="All Loans", command=lambda: self.refresh_loans_display()).pack()
        tk.Button(self.main_frame, text="Pending Loans", command=lambda: self.refresh_loans_display("Pending")).pack()
        tk.Button(self.main_frame, text="Accepted Loans", command=lambda: self.refresh_loans_display("Accepted")).pack()
        tk.Button(self.main_frame, text="Rejected Loans", command=lambda: self.refresh_loans_display("Rejected")).pack()
        self.refresh_loans_display()
        loans = Loan.select().where(Loan.user == self.current_user)
        for loan in loans:
            tk.Label(self.main_frame, text=f"Loan ID: {loan.id}, Amount: {loan.amount}, Status: {loan.status}, Repaid: {loan.repayment}").pack()
        tk.Button(self.main_frame, text="Back", command=self.user_dashboard).pack()
    def repay_loan_gui(self):
        self.clear_frame()
        # Add fields for loan repayment
        tk.Label(self.main_frame, text="Loan ID").pack()
        loan_id_entry = tk.Entry(self.main_frame)
        loan_id_entry.pack()

        tk.Label(self.main_frame, text="Repayment Amount").pack()
        repayment_amount_entry = tk.Entry(self.main_frame)
        repayment_amount_entry.pack()
        def on_repay_loan():
            loan_id = int(loan_id_entry.get())
            repayment_amount = float(repayment_amount_entry.get())
            loan = Loan.get_or_none(Loan.id == loan_id, Loan.user == self.current_user)
            if loan:
                loan.repayment += repayment_amount
                loan.save()
                messagebox.showinfo("Success", "Loan repayment recorded!")
            else:
             messagebox.showerror("Error", "Invalid loan ID or not authorized!")
        tk.Button(self.main_frame, text="Submit", command=on_repay_loan).pack()
        tk.Button(self.main_frame, text="Back", command=self.user_dashboard).pack()
def main():
    print("Starting the application...")
    setup_database()
    root = tk.Tk()
    app = App(root)
    root.mainloop()

if __name__ == "__main__":
     main()
