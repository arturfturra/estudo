import logging
import sqlite3
import tkinter as tk
from tkinter import ttk, messagebox
from ttkthemes import ThemedStyle
from bcrypt import checkpw, hashpw, gensalt
import pandas as pd
from reportlab.lib.pagesizes import letter
from reportlab.pdfgen import canvas
from openpyxl import Workbook

def configure_logging():
    logging.basicConfig(level=logging.INFO, format='%(asctime)s - %(levelname)s - %(message)s')

class RegistroApp:
    def __init__(self, root):
        self.root = root
        self.root.title("Registro de Entrada")
        self.root.geometry("800x600")
        self.root.configure(bg='#1f1f1f')
        self.root.resizable(False, False)

        style = ThemedStyle(self.root)
        style.set_theme("equilux")

        self.logged_in = False
        self.user_id = None
        self.username = None

        self.create_widgets()
        self.create_database()

    def create_widgets(self):
        self.login_frame = tk.Frame(self.root, bg='#1f1f1f')
        self.login_frame.pack(expand=True)

        self.username_label = tk.Label(self.login_frame, text="Usuário:", bg='#1f1f1f', fg='#cccccc')
        self.username_label.grid(row=0, column=0, padx=10, pady=5, sticky=tk.E)
        self.username_entry = tk.Entry(self.login_frame, bg='#333333', fg='#ffffff')
        self.username_entry.grid(row=0, column=1, padx=10, pady=5, sticky=tk.W+tk.E)

        self.password_label = tk.Label(self.login_frame, text="Senha:", bg='#1f1f1f', fg='#cccccc')
        self.password_label.grid(row=1, column=0, padx=10, pady=5, sticky=tk.E)
        self.password_entry = tk.Entry(self.login_frame, show="*", bg='#333333', fg='#ffffff')
        self.password_entry.grid(row=1, column=1, padx=10, pady=5, sticky=tk.W+tk.E)

        self.login_button = ttk.Button(self.login_frame, text="Login", command=self.login)
        self.login_button.grid(row=2, column=0, columnspan=2, padx=10, pady=5, sticky="we")

        self.create_user_button = ttk.Button(self.login_frame, text="Criar Usuário", command=self.create_user)
        self.create_user_button.grid(row=3, column=0, columnspan=2, padx=10, pady=5, sticky="we")

        self.status_label = tk.Label(self.root, text="", bg='#1f1f1f', fg='#cccccc')
        self.status_label.pack()

    def create_database(self):
        try:
            self.conn = sqlite3.connect('registro.db')
            self.cursor = self.conn.cursor()

            self.cursor.execute('''CREATE TABLE IF NOT EXISTS usuarios (
                                    id INTEGER PRIMARY KEY,
                                    username TEXT NOT NULL,
                                    password TEXT NOT NULL
                                )''')

            self.cursor.execute('''CREATE TABLE IF NOT EXISTS registros (
                                    id INTEGER PRIMARY KEY,
                                    user_id INTEGER,
                                    time TIMESTAMP,
                                    tipo TEXT,
                                    FOREIGN KEY(user_id) REFERENCES usuarios(id)
                                )''')
            logging.info("Banco de dados configurado com sucesso.")
        except sqlite3.Error as e:
            logging.error(f"Erro ao criar o banco de dados: {e}")

    def login(self):
        username = self.username_entry.get().strip()
        password = self.password_entry.get().strip()

        if not username or not password:
            self.status_label.config(text="Usuário e senha não podem estar vazios.")
            return

        try:
            logging.info(f"Tentativa de login com usuário: {username}")
            self.cursor.execute("SELECT id, username, password FROM usuarios WHERE username=?", (username,))
            user = self.cursor.fetchone()

            if user:
                logging.info(f"Usuário encontrado: {username}")
                if checkpw(password.encode(), user[2]):
                    self.logged_in = True
                    self.user_id = user[0]
                    self.username = user[1]
                    self.status_label.config(text=f"Bem-vindo, {self.username} (ID: {self.user_id})!")
                    logging.info(f"Usuário {self.username} logado com sucesso.")
                    self.show_register_page()
                else:
                    self.status_label.config(text="Usuário ou senha incorretos.")
                    logging.warning(f"Senha incorreta para o usuário {username}")
            else:
                self.status_label.config(text="Usuário ou senha incorretos.")
                logging.warning(f"Usuário {username} não encontrado.")
        except sqlite3.Error as e:
            logging.error(f"Erro ao fazer login: {e}")

    def create_user(self):
        username = self.username_entry.get().strip()
        password = self.password_entry.get().strip()

        if not username or not password:
            messagebox.showerror("Erro", "Usuário e senha não podem estar vazios.")
            return

        try:
            # Verifica se o usuário já existe
            self.cursor.execute("SELECT id FROM usuarios WHERE username=?", (username,))
            if self.cursor.fetchone():
                messagebox.showerror("Erro", "Usuário já existe.")
                return

            hashed_password = hashpw(password.encode(), gensalt())
            self.cursor.execute("INSERT INTO usuarios (username, password) VALUES (?, ?)", (username, hashed_password))
            self.conn.commit()
            messagebox.showinfo("Sucesso", "Usuário criado com sucesso!")
            logging.info(f"Usuário {username} criado com sucesso.")
        except sqlite3.Error as e:
            logging.error(f"Erro ao criar usuário: {e}")

    def show_register_page(self):
        self.login_frame.destroy()

        self.register_frame = tk.Frame(self.root, bg='#1f1f1f')
        self.register_frame.pack(expand=True, fill="both")

        self.register_table = ttk.Treeview(self.register_frame, columns=("#1", "#2"), show="headings")
        self.register_table.heading("#1", text="Horário")
        self.register_table.heading("#2", text="Tipo")
        self.register_table.pack(expand=True, fill="both", padx=10, pady=10)

        self.register_button_in = ttk.Button(self.register_frame, text="Registrar Entrada", command=lambda: self.register_time("Entrada"))
        self.register_button_in.pack(side="left", padx=10, pady=10)

        self.register_button_out = ttk.Button(self.register_frame, text="Registrar Saída", command=lambda: self.register_time("Saída"))
        self.register_button_out.pack(side="left", padx=10, pady=10)

        self.logout_button = ttk.Button(self.register_frame, text="Logout", command=self.logout)
        self.logout_button.pack(side="left", padx=10, pady=10)

        self.export_pdf_button = ttk.Button(self.register_frame, text="Extrair PDF", command=self.export_pdf)
        self.export_pdf_button.pack(side="left", padx=10, pady=10)

        self.export_excel_button = ttk.Button(self.register_frame, text="Extrair Excel", command=self.export_excel)
        self.export_excel_button.pack(side="left", padx=10, pady=10)

        self.update_register_table()

    def register_time(self, tipo):
        try:
            self.cursor.execute("INSERT INTO registros (user_id, time, tipo) VALUES (?, datetime('now'), ?)", (self.user_id, tipo))
            self.conn.commit()
            self.update_register_table()
            logging.info(f"Registro de {tipo} adicionado com sucesso.")
        except sqlite3.Error as e:
            logging.error(f"Erro ao registrar horário: {e}")

    def update_register_table(self):
        for row in self.register_table.get_children():
            self.register_table.delete(row)

        try:
            self.cursor.execute("SELECT time, tipo FROM registros WHERE user_id=?", (self.user_id,))
            for row in self.cursor.fetchall():
                self.register_table.insert("", "end", values=row)
        except sqlite3.Error as e:
            logging.error(f"Erro ao atualizar tabela de registros: {e}")

    def logout(self):
        self.logged_in = False
        self.user_id = None
        self.username = None
        self.status_label.config(text="Você foi desconectado.")
        self.register_frame.destroy()
        self.create_widgets()

    def export_pdf(self):
        try:
            file_name = f"registros_{self.username}_{self.user_id}.pdf"
            c = canvas.Canvas(file_name, pagesize=letter)
            c.drawString(100, 750, f"Relatório de Registros - {self.username} (ID: {self.user_id})")
            c.drawString(100, 730, "Horário                 | Tipo")
            y = 710
            self.cursor.execute("SELECT time, tipo FROM registros WHERE user_id=?", (self.user_id,))
            for row in self.cursor.fetchall():
                c.drawString(100, y, f"{row[0]} | {row[1]}")
                y -= 20
            c.save()
            messagebox.showinfo("Sucesso", f"PDF gerado com sucesso! Arquivo: {file_name}")
            logging.info(f"Relatório PDF gerado com sucesso: {file_name}")
        except Exception as e:
            logging.error(f"Erro ao gerar PDF: {e}")
            messagebox.showerror("Erro", "Erro ao gerar PDF.")

    def export_excel(self):
        try:
            file_name = f"registros_{self.username}_{self.user_id}.xlsx"
            wb = Workbook()
            ws = wb.active
            ws.append(["Horário", "Tipo"])

            self.cursor.execute("SELECT time, tipo FROM registros WHERE user_id=?", (self.user_id,))
            for row in self.cursor.fetchall():
                ws.append(row)

            wb.save(file_name)
            messagebox.showinfo("Sucesso", f"Excel gerado com sucesso! Arquivo: {file_name}")
            logging.info(f"Relatório Excel gerado com sucesso: {file_name}")
        except Exception as e:
            logging.error(f"Erro ao gerar Excel: {e}")
            messagebox.showerror("Erro", "Erro ao gerar Excel.")

if __name__ == "__main__":
    configure_logging()
    root = tk.Tk()
    app = RegistroApp(root)
    root.mainloop()
