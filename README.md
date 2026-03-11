## ⚡ GRVT Trading Bot

<p align="left">
  <img alt="GRVT trading bot" width="400px" height="auto" src="./screenshot.png">
</p>

## Table of content

* [Features](#features)
* [Prerequisites](#prerequisites)
* [Installation](#installation)
* [Usage](#usage)
* [How does it work](#how)

<a name="features"/>

## Features

* Automated trading bot for **GRVT exchange**
* Ready-to-run Python script
* Supports real-time market monitoring
* Fast order execution
* Simple one-file architecture
* Logging system for all trading activities
* Lightweight and easy to deploy

<a name="prerequisites"/>

## Prerequisites

* Python **3.8 – 3.11 recommended**

For Windows users: if `python` or `pip` isn't recognized as a command, make sure you installed Python with the option **"Add Python to PATH"** enabled.

<a name="installation"/>

## Installation

1. Clone the repository

```sh
git clone https://github.com/renjixyz/grvt-trading-bot
```

2. Go to the project directory

```sh
cd grvt-trading-bot
```

3. Install dependencies

```sh
pip install -r requirements.txt
```

4. Configure your API keys and trading parameters inside `main.py`

5. Run the bot

```sh
python bot_obf.py
```

<a name="usage"/>

## Usage

Run the bot with the following command:

```sh
python bot_obf.py 
```

This will start the bot in **test mode** using **500 USDT** on **ETH/USDT**.

<a name="how"/>

## How does it work?

The bot continuously monitors the market on **GRVT exchange** and automatically executes trades based on its internal strategy.

Basic workflow:

1. Fetch market data from GRVT
2. Analyze price movements
3. Detect trading opportunities
4. Execute buy/sell orders
5. Log all activity

The bot is designed to be lightweight and simple, running entirely from a **single Python file**.

---

⚠️ **Disclaimer**

Trading cryptocurrencies involves financial risk. Use this software at your own risk.
