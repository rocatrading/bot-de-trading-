import sys
from PyQt5.QtWidgets import QApplication, QMainWindow, QLabel, QVBoxLayout, QWidget, QLineEdit, QPushButton, QTextEdit
from web3 import Web3
from threading import Thread
import time
import requests

class TradingBot(QMainWindow):
    def __init__(self):
        super().__init__()
        self.initUI()
        self.web3 = Web3(Web3.HTTPProvider("https://mainnet.infura.io/v3/YOUR_INFURA_PROJECT_ID"))
        self.zapper_api_key = "YOUR_ZAPPER_API_KEY" # Remplacez par votre clé API Zapper

    def initUI(self):
        self.setWindowTitle("Trading Bot")
        
        self.central_widget = QWidget()
        self.setCentralWidget(self.central_widget)
        
        self.layout = QVBoxLayout(self.central_widget)
        
        self.wallet_label = QLabel("Adresse du portefeuille à suivre:")
        self.layout.addWidget(self.wallet_label)
        
        self.wallet_input = QLineEdit()
        self.layout.addWidget(self.wallet_input)
        
        self.start_button = QPushButton("Démarrer")
        self.start_button.clicked.connect(self.start_following)
        self.layout.addWidget(self.start_button)
        
        self.status_label = QLabel("Statut: En attente")
        self.layout.addWidget(self.status_label)

        # Section pour afficher les transactions en temps réel
        self.transactions_label = QLabel("Transactions en temps réel:")
        self.layout.addWidget(self.transactions_label)
        
        self.transactions_text = QTextEdit()
        self.transactions_text.setReadOnly(True)
        self.layout.addWidget(self.transactions_text)
        
        # Section pour les logs et l'historique
        self.logs_label = QLabel("Logs et Historique:")
        self.layout.addWidget(self.logs_label)
        
        self.logs_text = QTextEdit()
        self.logs_text.setReadOnly(True)
        self.layout.addWidget(self.logs_text)

    def start_following(self):
        wallet_address = self.wallet_input.text()
        self.status_label.setText(f"Suivi de {wallet_address}")
        self.get_zapper_info(wallet_address) # Nouvelle fonction pour obtenir des infos Zapper
        self.follow_wallet(wallet_address)

    def get_zapper_info(self, wallet_address):
        url = f"https://api.zapper.fi/v1/protocols/tokens/balances?addresses[]={wallet_address}&api_key={self.zapper_api_key}"
        response = requests.get(url)
        if response.status_code == 200:
            try:
                data = response.json()
                info = data[wallet_address]
                self.status_label.setText(f"Info Zapper: {info}")
                self.logs_text.append(f"Info Zapper: {info}")
            except requests.exceptions.JSONDecodeError:
                self.status_label.setText("Erreur de décodage JSON pour les informations Zapper")
        else:
            self.status_label.setText(f"Erreur HTTP {response.status_code} lors de la récupération des informations Zapper")

    def follow_wallet(self, wallet_address):
        def follow():
            filter = self.web3.eth.filter({"fromBlock": "latest", "address": wallet_address})
            while True:
                for tx in filter.get_new_entries():
                    self.process_transaction(tx)
                time.sleep(2) # Pause pour éviter une surcharge de la boucle

        thread = Thread(target=follow)
        thread.start()

    def process_transaction(self, tx):
        tx_details = f"Nouvelle transaction: {tx}"
        self.transactions_text.append(tx_details)
        self.logs_text.append(tx_details)
        self.status_label.setText(tx_details)

if __name__ == '__main__':
    app = QApplication(sys.argv)
    ex = TradingBot()
    ex.show()
    sys.exit(app.exec_())
