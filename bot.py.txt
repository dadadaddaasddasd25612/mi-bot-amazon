import requests
import json
import time
import re
import asyncio
import logging
from urllib.parse import urlencode, urljoin, quote
from bs4 import BeautifulSoup
import random
import uuid
import socket
from anticaptchaofficial.recaptchav2proxyless import recaptchaV2Proxyless
from anticaptchaofficial.imagecaptcha import imagecaptcha
import nest_asyncio
from playwright.async_api import async_playwright

# Aplicar nest_asyncio para bucles de eventos anidados
nest_asyncio.apply()

# Función para crear headers que simulen JavaScript habilitado
def create_js_headers(domain, user_agent=None):
    """Crea headers que simulan un navegador con JavaScript habilitado"""
    if not user_agent:
        user_agent = random.choice(USER_AGENTS)

    return {
        'Host': f'www.{domain}',
        'User-Agent': user_agent,
        'Accept': 'text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,image/apng,*/*;q=0.8,application/signed-exchange;v=b3;q=0.7',
        'Accept-Language': 'en-US,en;q=0.9,es;q=0.8',
        'Accept-Encoding': 'gzip, deflate, br',
        'DNT': '1',
        'Connection': 'keep-alive',
        'Upgrade-Insecure-Requests': '1',
        'Sec-Fetch-Dest': 'document',
        'Sec-Fetch-Mode': 'navigate',
        'Sec-Fetch-Site': 'none',
        'Sec-Fetch-User': '?1',
        'Cache-Control': 'max-age=0',
        'sec-ch-ua': '"Google Chrome";v="119", "Chromium";v="119", "Not?A_Brand";v="24"',
        'sec-ch-ua-mobile': '?0',
        'sec-ch-ua-platform': '"Windows"',
        'Sec-Purpose': 'prefetch'
    }

# Configurar logging
logging.basicConfig(
    filename='amazon_account_creator.log',
    level=logging.DEBUG,
    format='%(asctime)s - %(levelname)s - %(message)s'
)

# Configuración inicial
base_urls = {
    'amazon.ca': 'https://www.amazon.ca',
    'amazon.com.mx': 'https://www.amazon.com.mx',
    'amazon.com': 'https://www.amazon.com',
    'amazon.co.uk': 'https://www.amazon.co.uk',
    'amazon.de': 'https://www.amazon.de',
    'amazon.fr': 'https://www.amazon.fr',
    'amazon.it': 'https://www.amazon.it',
    'amazon.es': 'https://www.amazon.es',
    'amazon.co.jp': 'https://www.amazon.co.jp',
    'amazon.com.au': 'https://www.amazon.com.au',
    'amazon.in': 'https://www.amazon.in'
}
domains = [
    'amazon.ca', 'amazon.com.mx', 'amazon.com', 'amazon.co.uk', 'amazon.de',
    'amazon.fr', 'amazon.it', 'amazon.es', 'amazon.co.jp', 'amazon.com.au', 'amazon.in'
]
login_urls = {
    'amazon.ca': (
        "https://www.amazon.ca/ap/signin?ie=UTF8&openid.pape.max_auth_age=0&"
        "openid.return_to=https%3A%2F%2Fwww.amazon.ca%2F%3F_encoding%3DUTF8%26ref_%3Dnavm_accountmenu_switchacct&"
        "openid.identity=http%3A%2F%2Fspecs.openid.net%2Fauth%2F2.0%2Fidentifier_select&"
        "openid.assoc_handle=anywhere_v2_ca&_encoding=UTF8&openid.mode=checkid_setup&"
        "openid.claimed_id=http%3A%2F%2Fspecs.openid.net%2Fauth%2F2.0%2Fidentifier_select&"
        "openid.ns=http%3A%2F%2Fspecs.openid.net%2Fauth%2F2.0&switch_account=signin&"
        "ignoreAuthState=1&disableLoginPrepopulate=1&ref_=ap_sw_aa"
    ),
    'amazon.com.mx': (
        "https://www.amazon.com.mx/ap/signin?ie=UTF8&openid.pape.max_auth_age=0&"
        "openid.return_to=https%3A%2F%2Fwww.amazon.com.mx%2F%3F_encoding%3DUTF8%26ref_%3Dnavm_accountmenu_switchacct&"
        "openid.identity=http%3A%2F%2Fspecs.openid.net%2Fauth%2F2.0%2Fidentifier_select&"
        "openid.assoc_handle=anywhere_v2_mx&_encoding=UTF8&openid.mode=checkid_setup&"
        "openid.claimed_id=http%3A%2F%2Fspecs.openid.net%2Fauth%2F2.0%2Fidentifier_select&"
        "openid.ns=http%3A%2F%2Fspecs.openid.net%2Fauth%2F2.0&switch_account=signin&"
        "ignoreAuthState=1&disableLoginPrepopulate=1&ref_=ap_sw_aa"
    ),
    'amazon.com': (
        "https://www.amazon.com/ap/signin?ie=UTF8&openid.pape.max_auth_age=0&"
        "openid.return_to=https%3A%2F%2Fwww.amazon.com%2F%3F_encoding%3DUTF8%26ref_%3Dnavm_accountmenu_switchacct&"
        "openid.identity=http%3A%2F%2Fspecs.openid.net%2Fauth%2F2.0%2Fidentifier_select&"
        "openid.assoc_handle=anywhere_v2_us&_encoding=UTF8&openid.mode=checkid_setup&"
        "openid.claimed_id=http%3A%2F%2Fspecs.openid.net%2Fauth%2F2.0%2Fidentifier_select&"
        "openid.ns=http%3A%2F%2Fspecs.openid.net%2Fauth%2F2.0&switch_account=signin&"
        "ignoreAuthState=1&disableLoginPrepopulate=1&ref_=ap_sw_aa"
    ),
    'amazon.co.uk': (
        "https://www.amazon.co.uk/ap/signin?ie=UTF8&openid.pape.max_auth_age=0&"
        "openid.return_to=https%3A%2F%2Fwww.amazon.co.uk%2F%3F_encoding%3DUTF8%26ref_%3Dnavm_accountmenu_switchacct&"
        "openid.identity=http%3A%2F%2Fspecs.openid.net%2Fauth%2F2.0%2Fidentifier_select&"
        "openid.assoc_handle=anywhere_v2_uk&_encoding=UTF8&openid.mode=checkid_setup&"
        "openid.claimed_id=http%3A%2F%2Fspecs.openid.net%2Fauth%2F2.0%2Fidentifier_select&"
        "openid.ns=http%3A%2F%2Fspecs.openid.net%2Fauth%2F2.0&switch_account=signin&"
        "ignoreAuthState=1&disableLoginPrepopulate=1&ref_=ap_sw_aa"
    ),
    'amazon.de': (
        "https://www.amazon.de/ap/signin?ie=UTF8&openid.pape.max_auth_age=0&"
        "openid.return_to=https%3A%2F%2Fwww.amazon.de%2F%3F_encoding%3DUTF8%26ref_%3Dnavm_accountmenu_switchacct&"
        "openid.identity=http%3A%2F%2Fspecs.openid.net%2Fauth%2F2.0%2Fidentifier_select&"
        "openid.assoc_handle=anywhere_v2_de&_encoding=UTF8&openid.mode=checkid_setup&"
        "openid.claimed_id=http%3A%2F%2Fspecs.openid.net%2Fauth%2F2.0%2Fidentifier_select&"
        "openid.ns=http%3A%2F%2Fspecs.openid.net%2Fauth%2F2.0&switch_account=signin&"
        "ignoreAuthState=1&disableLoginPrepopulate=1&ref_=ap_sw_aa"
    ),
    'amazon.fr': (
        "https://www.amazon.fr/ap/signin?ie=UTF8&openid.pape.max_auth_age=0&"
        "openid.return_to=https%3A%2F%2Fwww.amazon.fr%2F%3F_encoding%3DUTF8%26ref_%3Dnavm_accountmenu_switchacct&"
        "openid.identity=http%3A%2F%2Fspecs.openid.net%2Fauth%2F2.0%2Fidentifier_select&"
        "openid.assoc_handle=anywhere_v2_fr&_encoding=UTF8&openid.mode=checkid_setup&"
        "openid.claimed_id=http%3A%2F%2Fspecs.openid.net%2Fauth%2F2.0%2Fidentifier_select&"
        "openid.ns=http%3A%2F%2Fspecs.openid.net%2Fauth%2F2.0&switch_account=signin&"
        "ignoreAuthState=1&disableLoginPrepopulate=1&ref_=ap_sw_aa"
    ),
    'amazon.it': (
        "https://www.amazon.it/ap/signin?ie=UTF8&openid.pape.max_auth_age=0&"
        "openid.return_to=https%3A%2F%2Fwww.amazon.it%2F%3F_encoding%3DUTF8%26ref_%3Dnavm_accountmenu_switchacct&"
        "openid.identity=http%3A%2F%2Fspecs.openid.net%2Fauth%2F2.0%2Fidentifier_select&"
        "openid.assoc_handle=anywhere_v2_it&_encoding=UTF8&openid.mode=checkid_setup&"
        "openid.claimed_id=http%3A%2F%2Fspecs.openid.net%2Fauth%2F2.0%2Fidentifier_select&"
        "openid.ns=http%3A%2F%2Fspecs.openid.net%2Fauth%2F2.0&switch_account=signin&"
        "ignoreAuthState=1&disableLoginPrepopulate=1&ref_=ap_sw_aa"
    ),
    'amazon.es': (
        "https://www.amazon.es/ap/signin?ie=UTF8&openid.pape.max_auth_age=0&"
        "openid.return_to=https%3A%2F%2Fwww.amazon.es%2F%3F_encoding%3DUTF8%26ref_%3Dnavm_accountmenu_switchacct&"
        "openid.identity=http%3A%2F%2Fspecs.openid.net%2Fauth%2F2.0%2Fidentifier_select&"
        "openid.assoc_handle=anywhere_v2_es&_encoding=UTF8&openid.mode=checkid_setup&"
        "openid.claimed_id=http%3A%2F%2Fspecs.openid.net%2Fauth%2F2.0%2Fidentifier_select&"
        "openid.ns=http%3A%2F%2Fspecs.openid.net%2Fauth%2F2.0&switch_account=signin&"
        "ignoreAuthState=1&disableLoginPrepopulate=1&ref_=ap_sw_aa"
    ),
    'amazon.co.jp': (
        "https://www.amazon.co.jp/ap/signin?ie=UTF8&openid.pape.max_auth_age=0&"
        "openid.return_to=https%3A%2F%2Fwww.amazon.co.jp%2F%3F_encoding%3DUTF8%26ref_%3Dnavm_accountmenu_switchacct&"
        "openid.identity=http%3A%2F%2Fspecs.openid.net%2Fauth%2F2.0%2Fidentifier_select&"
        "openid.assoc_handle=anywhere_v2_jp&_encoding=UTF8&openid.mode=checkid_setup&"
        "openid.claimed_id=http%3A%2F%2Fspecs.openid.net%2Fauth%2F2.0%2Fidentifier_select&"
        "openid.ns=http%3A%2F%2Fspecs.openid.net%2Fauth%2F2.0&switch_account=signin&"
        "ignoreAuthState=1&disableLoginPrepopulate=1&ref_=ap_sw_aa"
    ),
    'amazon.com.au': (
        "https://www.amazon.com.au/ap/signin?ie=UTF8&openid.pape.max_auth_age=0&"
        "openid.return_to=https%3A%2F%2Fwww.amazon.com.au%2F%3F_encoding%3DUTF8%26ref_%3Dnavm_accountmenu_switchacct&"
        "openid.identity=http%3A%2F%2Fspecs.openid.net%2Fauth%2F2.0%2Fidentifier_select&"
        "openid.assoc_handle=anywhere_v2_au&_encoding=UTF8&openid.mode=checkid_setup&"
        "openid.claimed_id=http%3A%2F%2Fspecs.openid.net%2Fauth%2F2.0%2Fidentifier_select&"
        "openid.ns=http%3A%2F%2Fspecs.openid.net%2Fauth%2F2.0&switch_account=signin&"
        "ignoreAuthState=1&disableLoginPrepopulate=1&ref_=ap_sw_aa"
    ),
    'amazon.in': (
        "https://www.amazon.in/ap/signin?ie=UTF8&openid.pape.max_auth_age=0&"
        "openid.return_to=https%3A%2F%2Fwww.amazon.in%2F%3F_encoding%3DUTF8%26ref_%3Dnavm_accountmenu_switchacct&"
        "openid.identity=http%3A%2F%2Fspecs.openid.net%2Fauth%2F2.0%2Fidentifier_select&"
        "openid.assoc_handle=anywhere_v2_in&_encoding=UTF8&openid.mode=checkid_setup&"
        "openid.claimed_id=http%3A%2F%2Fspecs.openid.net%2Fauth%2F2.0%2Fidentifier_select&"
        "openid.ns=http%3A%2F%2Fspecs.openid.net%2Fauth%2F2.0&switch_account=signin&"
        "ignoreAuthState=1&disableLoginPrepopulate=1&ref_=ap_sw_aa"
    )
}
address_book_urls = {
    'amazon.ca': "https://www.amazon.ca/a/addresses?ref_=ya_d_c_addr",
    'amazon.com.mx': "https://www.amazon.com.mx/a/addresses?ref_=ya_d_c_addr",
    'amazon.com': "https://www.amazon.com/a/addresses?ref_=ya_d_c_addr",
    'amazon.co.uk': "https://www.amazon.co.uk/a/addresses?ref_=ya_d_c_addr",
    'amazon.de': "https://www.amazon.de/a/addresses?ref_=ya_d_c_addr",
    'amazon.fr': "https://www.amazon.fr/a/addresses?ref_=ya_d_c_addr",
    'amazon.it': "https://www.amazon.it/a/addresses?ref_=ya_d_c_addr",
    'amazon.es': "https://www.amazon.es/a/addresses?ref_=ya_d_c_addr",
    'amazon.co.jp': "https://www.amazon.co.jp/a/addresses?ref_=ya_d_c_addr",
    'amazon.com.au': "https://www.amazon.com.au/a/addresses?ref_=ya_d_c_addr",
    'amazon.in': "https://www.amazon.in/a/addresses?ref_=ya_d_c_addr"
}
add_address_urls = {
    'amazon.ca': "https://www.amazon.ca/a/addresses/add?ref=ya_address_book_add_button",
    'amazon.com.mx': "https://www.amazon.com.mx/a/addresses/add?ref=ya_address_book_add_button",
    'amazon.com': "https://www.amazon.com/a/addresses/add?ref=ya_address_book_add_button",
    'amazon.co.uk': "https://www.amazon.co.uk/a/addresses/add?ref=ya_address_book_add_button",
    'amazon.de': "https://www.amazon.de/a/addresses/add?ref=ya_address_book_add_button",
    'amazon.fr': "https://www.amazon.fr/a/addresses/add?ref=ya_address_book_add_button",
    'amazon.it': "https://www.amazon.it/a/addresses/add?ref=ya_address_book_add_button",
    'amazon.es': "https://www.amazon.es/a/addresses/add?ref=ya_address_book_add_button",
    'amazon.co.jp': "https://www.amazon.co.jp/a/addresses/add?ref=ya_address_book_add_button",
    'amazon.com.au': "https://www.amazon.com.au/a/addresses/add?ref=ya_address_book_add_button",
    'amazon.in': "https://www.amazon.in/a/addresses/add?ref=ya_address_book_add_button"
}
wallet_urls = {
    'amazon.ca': "https://www.amazon.ca/cpe/yourpayments/wallet?ref_=ya_mb_mpo",
    'amazon.com.mx': "https://www.amazon.com.mx/cpe/yourpayments/wallet?ref_=ya_mb_mpo",
    'amazon.com': "https://www.amazon.com/cpe/yourpayments/wallet?ref_=ya_mb_mpo",
    'amazon.co.uk': "https://www.amazon.co.uk/cpe/yourpayments/wallet?ref_=ya_mb_mpo",
    'amazon.de': "https://www.amazon.de/cpe/yourpayments/wallet?ref_=ya_mb_mpo",
    'amazon.fr': "https://www.amazon.fr/cpe/yourpayments/wallet?ref_=ya_mb_mpo",
    'amazon.it': "https://www.amazon.it/cpe/yourpayments/wallet?ref_=ya_mb_mpo",
    'amazon.es': "https://www.amazon.es/cpe/yourpayments/wallet?ref_=ya_mb_mpo",
    'amazon.co.jp': "https://www.amazon.co.jp/cpe/yourpayments/wallet?ref_=ya_mb_mpo",
    'amazon.com.au': "https://www.amazon.com.au/cpe/yourpayments/wallet?ref_=ya_mb_mpo",
    'amazon.in': "https://www.amazon.in/cpe/yourpayments/wallet?ref_=ya_mb_mpo"
}
proxy = "p.webshare.io:80"
proxy_auth = "vgdgihxr-rotate:czeted9ynghb"
temp_mail_api = "https://api.mail.tm"
fallback_temp_mail_api = "https://api.guerrillamail.com"
second_fallback_temp_mail_api = "https://api.tempmail.plus"
ANTI_CAPTCHA_KEY = "37f2bc34098021f0fcb8ed61cc7b3782"

# Lista de User-Agents para rotación
USER_AGENTS = [
    'Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/135.0.0.0 Safari/537.36',
    'Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/135.0.0.0 Safari/537.36',
    'Mozilla/5.0 (Windows NT 10.0; Win64; x64; rv:130.0) Gecko/20100101 Firefox/130.0',
    'Mozilla/5.0 (Macintosh; Intel Mac OS X 10.15; rv:130.0) Gecko/20100101 Firefox/130.0'
]

# Cookies iniciales preestablecidas por región
INITIAL_COOKIES = {
    'amazon.ca': {
        'session-id': '137-5066096-0222707',
        'i18n-prefs': 'CAD',
        'lc-acbca': 'en_CA',
        'ubid-acbca': '130-5735560-8198653',
        'session-id-time': '2390080760l',
        'session-token': '46OjLiHW5IOx13iDkR2eadYh7aKCNec0d2oHC0Jb2Fr4CYCFlMWAiWwguMsCsp+5mEoqzc5zxCjZCtJusev+gZJhsTC2L4w3HzXSGWbyx8q89sGvopvJkgo8alP/tDPP/oj8kZ8GYZRRjU6vh9q15fE7ygY29958aAoCHfRxlDh8JTfI6kyOiM3OrDz8k7lAcXXp5pIdzWBRHpOkLznTQtj0+CqAWr9eYAdL1Gx4jhNCIGPqZKchKhrk7Ef4uhaszqzUKRS2ogmB2sqIo6SifsRuN0M6LgoSGvpI4irSH/x01Sm2TdjdCU9CFBAJ3YsbRjnLARR3WA+2DPlXf5+ReFY3wCeNiCpF',
        'csm-hit': '0JD9XD2CZTBSJ6WD833H+s-0JD9XD2CZTBSJ6WD833H|1759360807690',
        'rxc': 'AEmw0WhJORbNskYNwk0'
    },
    'amazon.com.mx': {
        'session-id': '147-1234567-8901234',
        'i18n-prefs': 'MXN',
        'lc-acbmx': 'es_MX',
        'ubid-acbmx': '140-9876543-2109876',
        'session-id-time': '2390080760l',
        'session-token': 'TBD',
        'csm-hit': 'TBD',
        'rxc': 'TBD'
    },
    'amazon.com': {
        'session-id': '134-9876543-2109876',
        'i18n-prefs': 'USD',
        'lc-main': 'en_US',
        'ubid-main': '150-1234567-8901234',
        'session-id-time': '2390080760l',
        'session-token': 'TBD',
        'csm-hit': 'TBD',
        'rxc': 'TBD'
    },
    'amazon.co.uk': {
        'session-id': '258-1234567-8901234',
        'i18n-prefs': 'GBP',
        'lc-acbuk': 'en_GB',
        'ubid-acbuk': '260-9876543-2109876',
        'session-id-time': '2390080760l',
        'session-token': 'TBD',
        'csm-hit': 'TBD',
        'rxc': 'TBD'
    },
    'amazon.de': {
        'session-id': '262-1234567-8901234',
        'i18n-prefs': 'EUR',
        'lc-acbde': 'de_DE',
        'ubid-acbde': '264-9876543-2109876',
        'session-id-time': '2390080760l',
        'session-token': 'TBD',
        'csm-hit': 'TBD',
        'rxc': 'TBD'
    },
    'amazon.fr': {
        'session-id': '266-1234567-8901234',
        'i18n-prefs': 'EUR',
        'lc-acbfr': 'fr_FR',
        'ubid-acbfr': '268-9876543-2109876',
        'session-id-time': '2390080760l',
        'session-token': 'TBD',
        'csm-hit': 'TBD',
        'rxc': 'TBD'
    },
    'amazon.it': {
        'session-id': '270-1234567-8901234',
        'i18n-prefs': 'EUR',
        'lc-acbit': 'it_IT',
        'ubid-acbit': '272-9876543-2109876',
        'session-id-time': '2390080760l',
        'session-token': 'TBD',
        'csm-hit': 'TBD',
        'rxc': 'TBD'
    },
    'amazon.es': {
        'session-id': '274-1234567-8901234',
        'i18n-prefs': 'EUR',
        'lc-acbes': 'es_ES',
        'ubid-acbes': '276-9876543-2109876',
        'session-id-time': '2390080760l',
        'session-token': 'TBD',
        'csm-hit': 'TBD',
        'rxc': 'TBD'
    },
    'amazon.co.jp': {
        'session-id': '278-1234567-8901234',
        'i18n-prefs': 'JPY',
        'lc-acbjp': 'ja_JP',
        'ubid-acbjp': '280-9876543-2109876',
        'session-id-time': '2390080760l',
        'session-token': 'TBD',
        'csm-hit': 'TBD',
        'rxc': 'TBD'
    },
    'amazon.com.au': {
        'session-id': '282-1234567-8901234',
        'i18n-prefs': 'AUD',
        'lc-acbau': 'en_AU',
        'ubid-acbau': '284-9876543-2109876',
        'session-id-time': '2390080760l',
        'session-token': 'TBD',
        'csm-hit': 'TBD',
        'rxc': 'TBD'
    },
    'amazon.in': {
        'session-id': '286-1234567-8901234',
        'i18n-prefs': 'INR',
        'lc-acbin': 'en_IN',
        'ubid-acbin': '288-9876543-2109876',
        'session-id-time': '2390080760l',
        'session-token': 'TBD',
        'csm-hit': 'TBD',
        'rxc': 'TBD'
    }
}

# Función para resolver CAPTCHA usando Anti-Captcha
def solve_captcha(site_key, url, is_image_captcha=False, image_url=None):
    try:
        if is_image_captcha and image_url:
            solver = imagecaptcha()
            solver.set_api_key(ANTI_CAPTCHA_KEY)
            solver.set_verbose(1)
            captcha_solution = solver.solve_and_return_solution(image_url)
            if captcha_solution:
                logging.info(f"Image CAPTCHA resuelto: {captcha_solution}")
                return captcha_solution
            else:
                logging.error("No se pudo resolver el CAPTCHA de imagen")
                return None
        else:
            solver = recaptchaV2Proxyless()
            solver.set_api_key(ANTI_CAPTCHA_KEY)
            solver.set_website_url(url)
            solver.set_website_key(site_key)
            solver.set_verbose(1)
            captcha_solution = solver.solve_and_return_solution()
            if captcha_solution:
                logging.info(f"reCAPTCHA v2 resuelto: {captcha_solution}")
                return captcha_solution
            else:
                logging.error("No se pudo resolver el reCAPTCHA v2")
                return None
    except Exception as e:
        logging.error(f"Error al resolver CAPTCHA: {str(e)}")
        return None

# Función para probar el proxy
def test_proxy(session):
    try:
        response = session.get('https://api.ipify.org?format=json', timeout=15)
        logging.info(f"**Respuesta de prueba del proxy**: {response.status_code} - {response.text[:500]}")
        
        if response.status_code != 200:
            logging.error(f"Proxy test failed with status code: {response.status_code}")
            return False, f"Status code {response.status_code}: {response.text[:500]}"
        
        if 'application/json' not in response.headers.get('Content-Type', ''):
            logging.error(f"Proxy test response is not JSON: {response.text[:500]}")
            return False, f"Non-JSON response: {response.text[:500]}"
        
        try:
            data = response.json()
            return True, data['ip']
        except json.JSONDecodeError as e:
            logging.error(f"JSON decode error: {str(e)} - Response: {response.text[:500]}")
            return False, f"JSON decode error: {str(e)}"
    except requests.exceptions.RequestException as e:
        logging.error(f"Proxy test failed: {str(e)}")
        return False, str(e)

# Función para verificar resolución DNS
def check_dns(hostname):
    try:
        socket.gethostbyname(hostname)
        logging.info(f"**DNS resuelto para {hostname}**")
        return True
    except socket.gaierror as e:
        logging.error(f"Error de resolución DNS para {hostname}: {str(e)}")
        return False

# Función para extraer texto entre dos cadenas
def get_str(string, start, end, occurrence=1):
    try:
        pattern = f'{re.escape(start)}(.*?){re.escape(end)}'
        matches = re.finditer(pattern, string)
        for i, match in enumerate(matches, 1):
            if i == occurrence:
                return match.group(1)
        return None
    except:
        return None

# Función para convertir cookies según el país
def convert_cookie(cookie_str, output_format='CA'):
    country_codes = {
        'CA': {'code': 'acbca', 'currency': 'CAD', 'lc': 'lc-acbca', 'lc_value': 'en_CA'},
        'MX': {'code': 'acbmx', 'currency': 'MXN', 'lc': 'lc-acbmx', 'lc_value': 'es_MX'},
        'US': {'code': 'main', 'currency': 'USD', 'lc': 'lc-main', 'lc_value': 'en_US'},
        'UK': {'code': 'acbuk', 'currency': 'GBP', 'lc': 'lc-acbuk', 'lc_value': 'en_GB'},
        'DE': {'code': 'acbde', 'currency': 'EUR', 'lc': 'lc-acbde', 'lc_value': 'de_DE'},
        'FR': {'code': 'acbfr', 'currency': 'EUR', 'lc': 'lc-acbfr', 'lc_value': 'fr_FR'},
        'IT': {'code': 'acbit', 'currency': 'EUR', 'lc': 'lc-acbit', 'lc_value': 'it_IT'},
        'ES': {'code': 'acbes', 'currency': 'EUR', 'lc': 'lc-acbes', 'lc_value': 'es_ES'},
        'JP': {'code': 'acbjp', 'currency': 'JPY', 'lc': 'lc-acbjp', 'lc_value': 'ja_JP'},
        'AU': {'code': 'acbau', 'currency': 'AUD', 'lc': 'lc-acbau', 'lc_value': 'en_AU'},
        'IN': {'code': 'acbin', 'currency': 'INR', 'lc': 'lc-acbin', 'lc_value': 'en_IN'}
    }
    if output_format not in country_codes:
        return cookie_str
    country = country_codes[output_format]
    cookie_str = re.sub(r'acbes|acbmx|acbit|main|acbca|acbde|acbuk|acbau|acbjp|acbfr|acbin', country['code'], cookie_str)
    cookie_str = re.sub(r'(i18n-prefs=)[A-Z]{3}', r'\1' + country['currency'], cookie_str)
    cookie_str = re.sub(rf'({country["lc"]}=)[a-z]{{2}}_[A-Z]{{2}}', r'\1' + country['lc_value'], cookie_str)
    return cookie_str

# Función para generar correo temporal
async def generate_temp_email():
    services = [
        ('mail.tm', temp_mail_api, lambda data: data.get('hydra:member', [])),
        ('guerrillamail', fallback_temp_mail_api, lambda data: [data]),
        ('tempmail.plus', second_fallback_temp_mail_api, lambda data: [data])
    ]
    
    for service_name, api_url, parse_domains in services:
        if not check_dns(api_url.replace('https://', '')):
            print(f"**❌ Error de DNS para {service_name}**")
            continue
        
        for attempt in range(5):
            try:
                if service_name == 'mail.tm':
                    response = requests.get(f"{api_url}/domains", timeout=15)
                    if response.status_code == 429:
                        print(f"**⚠️ Error 429: Demasiadas solicitudes en {service_name}, reintentando...**")
                        await asyncio.sleep(4 ** attempt)
                        continue
                    if response.status_code != 200:
                        continue
                    data = response.json()
                    domains = parse_domains(data)
                    if not domains or not domains[0].get('domain'):
                        continue
                    domain = domains[0]['domain']
                    email = f"{uuid.uuid4().hex[:8]}@{domain}"
                    password = f"Pass{random.randint(1000, 9999)}{uuid.uuid4().hex[:8]}"
                    payload = {"address": email, "password": password}
                    response = requests.post(f"{api_url}/accounts", json=payload, timeout=15)
                    if response.status_code == 201:
                        token = response.json().get('token')
                        print(f"**📧 Correo generado**: {email} 🟢")
                        return email, token, service_name
                elif service_name == 'guerrillamail':
                    response = requests.get(f"{api_url}/ajax.php?f=get_email_address&ip=127.0.0.1", timeout=15)
                    if response.status_code == 429:
                        print(f"**⚠️ Error 429: Demasiadas solicitudes en {service_name}, reintentando...**")
                        await asyncio.sleep(4 ** attempt)
                        continue
                    if response.status_code == 200:
                        data = response.json()
                        email = data.get('email_addr')
                        token = data.get('sid_token')
                        if email and token:
                            print(f"**📧 Correo generado**: {email} 🟢")
                            return email, token, service_name
                elif service_name == 'tempmail.plus':
                    response = requests.get(f"{api_url}/generate", timeout=15)
                    if response.status_code == 429:
                        print(f"**⚠️ Error 429: Demasiadas solicitudes en {service_name}, reintentando...**")
                        await asyncio.sleep(4 ** attempt)
                        continue
                    if response.status_code == 200:
                        data = response.json()
                        email = data.get('email')
                        token = data.get('token')
                        if email and token:
                            print(f"**📧 Correo generado**: {email} 🟢")
                            return email, token, service_name
            except Exception as e:
                logging.error(f"Error en {service_name}: {str(e)}")
                await asyncio.sleep(4 ** attempt)
    
    print("**❌ No se pudo generar correo**")
    return None, None, None

# Función para obtener el código de verificación del correo
async def get_verification_code(email, token, service, max_attempts=15, wait_time=10):
    try:
        for _ in range(max_attempts):
            if service == 'mail.tm' and token:
                response = requests.get(
                    f"{temp_mail_api}/messages",
                    headers={'Authorization': f'Bearer {token}'},
                    timeout=15
                )
                if response.status_code == 200:
                    emails = response.json().get('hydra:member', [])
                    for mail in emails:
                        verification_code = get_str(mail.get('content', ''), 'Your verification code is ', '\n')
                        if verification_code:
                            logging.info(f"Código de verificación encontrado: {verification_code}")
                            print(f"**✅ Código de verificación**: {verification_code} 🟢")
                            return verification_code
            elif service == 'guerrillamail' and token:
                response = requests.get(
                    f"{fallback_temp_mail_api}/ajax.php?f=check_email&seq=0&sid_token={token}",
                    timeout=15
                )
                if response.status_code == 200:
                    emails = response.json().get('list', [])
                    for mail in emails:
                        verification_code = get_str(mail.get('mail_body', ''), 'Your verification code is ', '\n')
                        if verification_code:
                            logging.info(f"Código de verificación encontrado: {verification_code}")
                            print(f"**✅ Código de verificación**: {verification_code} 🟢")
                            return verification_code
            elif service == 'tempmail.plus' and token:
                response = requests.get(
                    f"{second_fallback_temp_mail_api}/messages/{token}",
                    timeout=15
                )
                if response.status_code == 200:
                    emails = response.json().get('messages', [])
                    for mail in emails:
                        verification_code = get_str(mail.get('text', ''), 'Your verification code is ', '\n')
                        if verification_code:
                            logging.info(f"Código de verificación encontrado: {verification_code}")
                            print(f"**✅ Código de verificación**: {verification_code} 🟢")
                            return verification_code
            await asyncio.sleep(wait_time)
        logging.error("No se recibió el código de verificación")
        print("**❌ No se recibió el código de verificación**")
        return None
    except Exception as e:
        logging.error(f"Error al obtener código de verificación: {str(e)}")
        print(f"**❌ Error al obtener código de verificación**: {str(e)}")
        return None

# Función para iniciar sesión nuevamente si es necesario
async def login_again(session, domain, email, password, token=None, service=None, max_attempts=3):
    for attempt in range(max_attempts):
        try:
            headers = {
                'Host': f'www.{domain}',
                'User-Agent': random.choice(USER_AGENTS),
                'Accept': 'text/html,application/xhtml+xml,application/xml;q=0.9,image/webp,*/*;q=0.8',
                'Accept-Language': 'en-US,en;q=0.9,es-MX;q=0.8,fr-FR;q=0.7,de-DE;q=0.7,it-IT;q=0.7,es-ES;q=0.7,ja-JP;q=0.7',
                'Connection': 'keep-alive',
                'Upgrade-Insecure-Requests': '1',
                'Sec-Fetch-Dest': 'document',
                'Sec-Fetch-Mode': 'navigate',
                'Sec-Fetch-Site': 'same-origin',
                'Sec-Fetch-User': '?1'
            }
            # Limpiar cookies irrelevantes para evitar conflictos
            session.cookies.clear_expired_cookies()
            # Asegurar cookies iniciales
            for key, value in INITIAL_COOKIES[domain].items():
                session.cookies.set(key, value, domain=f".{domain}")

            # Obtener la página de login
            response = session.get(login_urls[domain], headers=headers, timeout=15, allow_redirects=True)
            logging.info(f"**Intento {attempt+1}/{max_attempts} - Respuesta página de login ({domain})**: {response.status_code} - URL: {response.url}")
            with open(f'login_page_{email}_{domain}_attempt_{attempt+1}.html', 'w', encoding='utf-8') as f:
                f.write(response.text)

            if response.status_code != 200:
                logging.error(f"Fallo al acceder a la página de login para {email}: {response.status_code}")
                await asyncio.sleep(2 ** attempt)
                continue

            soup = BeautifulSoup(response.text, 'html.parser')
            # Verificar si hay CAPTCHA
            captcha_div = soup.find('div', {'id': 'captchacharacters'})
            recaptcha_div = soup.find('div', {'class': re.compile('g-recaptcha')})
            if 'captcha' in response.text.lower() or captcha_div or recaptcha_div:
                logging.info("CAPTCHA detectado en página de login")
                if recaptcha_div:
                    site_key = recaptcha_div.get('data-sitekey')
                    if site_key:
                        captcha_solution = solve_captcha(site_key, login_urls[domain])
                        if not captcha_solution:
                            logging.error("No se pudo resolver el reCAPTCHA")
                            await asyncio.sleep(2 ** attempt)
                            continue
                        payload = {'g-recaptcha-response': captcha_solution}
                        response = session.post(
                            login_urls[domain],
                            headers={
                                'Host': f'www.{domain}',
                                'Content-Type': 'application/x-www-form-urlencoded',
                                'Origin': base_urls[domain],
                                'Referer': login_urls[domain],
                                'Accept': 'text/html,application/xhtml+xml,application/xml;q=0.9,image/webp,*/*;q=0.8',
                                'Accept-Language': 'en-US,en;q=0.9,es-MX;q=0.8,fr-FR;q=0.7,de-DE;q=0.7,it-IT;q=0.7,es-ES;q=0.7,ja-JP;q=0.7'
                            },
                            data=urlencode(payload),
                            timeout=15,
                            allow_redirects=True
                        )
                        soup = BeautifulSoup(response.text, 'html.parser')
                        logging.info(f"**Respuesta tras resolver reCAPTCHA**: {response.status_code} - URL: {response.url}")
                    else:
                        logging.error("No se encontró sitekey para reCAPTCHA")
                        await asyncio.sleep(2 ** attempt)
                        continue
                elif captcha_div:
                    captcha_img = captcha_div.find('img')
                    if captcha_img and captcha_img.get('src'):
                        captcha_solution = solve_captcha(None, login_urls[domain], is_image_captcha=True, image_url=captcha_img['src'])
                        if not captcha_solution:
                            logging.error("No se pudo resolver el CAPTCHA de imagen")
                            await asyncio.sleep(2 ** attempt)
                            continue
                        payload = {'cvf_captcha_input': captcha_solution}
                        response = session.post(
                            login_urls[domain],
                            headers={
                                'Host': f'www.{domain}',
                                'Content-Type': 'application/x-www-form-urlencoded',
                                'Origin': base_urls[domain],
                                'Referer': login_urls[domain],
                                'Accept': 'text/html,application/xhtml+xml,application/xml;q=0.9,image/webp,*/*;q=0.8',
                                'Accept-Language': 'en-US,en;q=0.9,es-MX;q=0.8,fr-FR;q=0.7,de-DE;q=0.7,it-IT;q=0.7,es-ES;q=0.7,ja-JP;q=0.7'
                            },
                            data=urlencode(payload),
                            timeout=15,
                            allow_redirects=True
                        )
                        soup = BeautifulSoup(response.text, 'html.parser')
                        logging.info(f"**Respuesta tras resolver CAPTCHA de imagen**: {response.status_code} - URL: {response.url}")
                    else:
                        logging.error("No se encontró imagen de CAPTCHA")
                        await asyncio.sleep(2 ** attempt)
                        continue
                else:
                    logging.error("CAPTCHA detectado pero no se encontró reCAPTCHA ni imagen")
                    await asyncio.sleep(2 ** attempt)
                    continue

            # Buscar formulario de login
            form = (soup.find('form', {'id': re.compile('ap_signin_form|signin|login', re.I)}) or
                    soup.find('form', {'action': re.compile('signin|sign-in|login', re.I)}) or
                    soup.find('form', {'method': 'post'}))
            if not form:
                logging.error(f"No se encontró formulario de login en {domain}")
                await asyncio.sleep(2 ** attempt)
                continue

            inputs = form.find_all('input', type='hidden')
            payload = {inp.get('name'): inp.get('value') for inp in inputs if inp.get('name')}
            payload.update({
                'email': email,
                'password': password,
                'rememberMe': 'true'
            })
            form_action = urljoin(base_urls[domain], form.get('action') or login_urls[domain])

            response = session.post(
                form_action,
                headers={
                    'Host': f'www.{domain}',
                    'Content-Type': 'application/x-www-form-urlencoded',
                    'Origin': base_urls[domain],
                    'Referer': login_urls[domain],
                    'User-Agent': random.choice(USER_AGENTS),
                    'Accept': 'text/html,application/xhtml+xml,application/xml;q=0.9,image/webp,*/*;q=0.8',
                    'Accept-Language': 'en-US,en;q=0.9,es-MX;q=0.8,fr-FR;q=0.7,de-DE;q=0.7,it-IT;q=0.7,es-ES;q=0.7,ja-JP;q=0.7'
                },
                data=urlencode(payload),
                timeout=15,
                allow_redirects=True
            )
            logging.info(f"**Intento {attempt+1}/{max_attempts} - Respuesta de re-login ({domain})**: {response.status_code} - URL: {response.url}")
            with open(f'login_response_{email}_{domain}_attempt_{attempt+1}.html', 'w', encoding='utf-8') as f:
                f.write(response.text)

            soup = BeautifulSoup(response.text, 'html.parser')
            # Verificar si se requiere verificación en dos pasos
            if 'cvf' in response.url.lower() or 'verification' in response.text.lower():
                logging.info(f"Verificación en dos pasos detectada para {email}")
                verification_code = await get_verification_code(email, token, service)
                if not verification_code:
                    logging.error(f"No se pudo obtener el código de verificación para {email}")
                    await asyncio.sleep(2 ** attempt)
                    continue

                cvf_form = (soup.find('form', {'id': 'cvf_form'}) or
                            soup.find('form', {'action': re.compile('cvf|verify|verification', re.I)}) or
                            soup.find('form', {'method': 'post'}))
                if not cvf_form:
                    logging.error(f"No se encontró formulario de verificación en {domain}")
                    await asyncio.sleep(2 ** attempt)
                    continue

                cvf_payload = {inp.get('name'): inp.get('value') for inp in cvf_form.find_all('input', type='hidden') if inp.get('name')}
                cvf_payload.update({
                    'code': verification_code,
                    'appAction': 'VERIFY',
                    'appActionToken': get_str(response.text, 'name="appActionToken" value="', '"') or '',
                    'workflowState': get_str(response.text, 'name="workflowState" value="', '"') or ''
                })

                response = session.post(
                    urljoin(base_urls[domain], cvf_form.get('action') or response.url),
                    headers={
                        'Host': f'www.{domain}',
                        'Content-Type': 'application/x-www-form-urlencoded',
                        'Origin': base_urls[domain],
                        'Referer': response.url,
                        'Accept': 'text/html,application/xhtml+xml,application/xml;q=0.9,image/webp,*/*;q=0.8',
                        'Accept-Language': 'en-US,en;q=0.9,es-MX;q=0.8,fr-FR;q=0.7,de-DE;q=0.7,it-IT;q=0.7,es-ES;q=0.7,ja-JP;q=0.7'
                    },
                    data=urlencode(cvf_payload),
                    timeout=15,
                    allow_redirects=True
                )
                logging.info(f"**Respuesta verificación en dos pasos ({domain})**: {response.status_code} - URL: {response.url}")
                with open(f'login_verification_response_{email}_{domain}_attempt_{attempt+1}.html', 'w', encoding='utf-8') as f:
                    f.write(response.text)
                soup = BeautifulSoup(response.text, 'html.parser')

            # Verificar si el login fue exitoso
            if response.status_code == 200 and 'signin' not in response.url.lower() and 'authentication' not in response.text.lower():
                logging.info(f"Re-autenticación exitosa para {email}")
                cookies = session.cookies.get_dict()
                cookie_str = '; '.join([f"{key}={value}" for key, value in cookies.items()])
                logging.info(f"Cookies tras re-autenticación para {email}: {cookie_str}")
                return True
            
            error_div = soup.find('div', {'class': re.compile('a-alert-error|a-alert-warning|a-box-error', re.I)})
            if error_div:
                error_message = error_div.get_text(strip=True)
                logging.error(f"Error en re-login para {email}: {error_message}")
                await asyncio.sleep(2 ** attempt)
                continue

            logging.error(f"Fallo en re-autenticación para {email}. Status: {response.status_code}, URL: {response.url}")
            await asyncio.sleep(2 ** attempt)
            continue
        except Exception as e:
            logging.error(f"Intento {attempt+1}/{max_attempts} - Error al re-autenticar para {email}: {str(e)}")
            await asyncio.sleep(2 ** attempt)
            continue
    
    logging.error(f"Fallo en re-autenticación para {email} después de {max_attempts} intentos")
    return False

# Función para agregar dirección
async def add_address(session, domain, email, password, token=None, service=None):
    try:
        if domain not in base_urls:
            logging.error(f"Dominio inválido: {domain}")
            print(f"**❌ Dominio inválido: {domain}**")
            return None

        # Asegurar cookies iniciales
        for key, value in INITIAL_COOKIES[domain].items():
            session.cookies.set(key, value, domain=f".{domain}")

        cookies = session.cookies.get_dict()
        cookie_str = '; '.join([f"{key}={value}" for key, value in cookies.items()])
        logging.info(f"Cookies iniciales para {email} en {domain}: {cookie_str}")

        headers = {
            "Host": f"www.{domain}",
            "Referer": f"{base_urls[domain]}/gp/your-account?ref_=nav_AccountFlyout_ya",
            "User-Agent": random.choice(USER_AGENTS),
            "Accept": "text/html,application/xhtml+xml,application/xml;q=0.9,image/webp,*/*;q=0.8",
            "Accept-Language": 'en-US,en;q=0.9,es-MX;q=0.8,fr-FR;q=0.7,de-DE;q=0.7,it-IT;q=0.7,es-ES;q=0.7,ja-JP;q=0.7',
            "Viewport-Width": "1536"
        }

        # Acceder a la página de libreta de direcciones
        response = session.get(address_book_urls[domain], headers=headers, timeout=15, allow_redirects=True)
        logging.info(f"**Respuesta página de libreta de direcciones ({domain})**: {response.status_code} - URL: {response.url}")
        with open(f'address_book_page_{email}_{domain}.html', 'w', encoding='utf-8') as f:
            f.write(response.text)

        if response.status_code != 200:
            logging.error(f"Fallo al acceder a la página de libreta de direcciones para {email} en {domain}: {response.status_code}")
            print(f"**❌ Fallo al acceder a la página de libreta de direcciones para {email}**: {response.status_code}")
            return None

        # Verificar si se requiere re-autenticación
        if 'signin' in response.url.lower() or 'authentication' in response.text.lower():
            logging.info(f"Se requiere autenticación adicional para {email}. Intentando re-login...")
            if not await login_again(session, domain, email, password, token, service):
                logging.error(f"Fallo en re-autenticación para {email}")
                print(f"**❌ Fallo en re-autenticación para {email}**")
                return None
            response = session.get(address_book_urls[domain], headers=headers, timeout=15, allow_redirects=True)
            logging.info(f"**Respuesta página de libreta de direcciones tras re-login ({domain})**: {response.status_code} - URL: {response.url}")
            with open(f'address_book_page_{email}_{domain}_relogin.html', 'w', encoding='utf-8') as f:
                f.write(response.text)
            if response.status_code != 200:
                logging.error(f"Fallo al re-acceder a la página de libreta de direcciones para {email}: {response.status_code}")
                print(f"**❌ Fallo al re-acceder a la página de libreta de direcciones para {email}**: {response.status_code}")
                return None

        # Acceder a la página de añadir dirección
        response = session.get(
            add_address_urls[domain],
            headers=headers,
            timeout=15,
            allow_redirects=True
        )
        logging.info(f"**Respuesta página de adición de dirección ({domain})**: {response.status_code} - URL: {response.url}")
        with open(f'address_page_{email}_{domain}.html', 'w', encoding='utf-8') as f:
            f.write(response.text)

        if response.status_code != 200:
            logging.error(f"Fallo al acceder a la página de adición de dirección para {email} en {domain}: {response.status_code}")
            print(f"**❌ Fallo al acceder a la página de adición de dirección para {email}**: {response.status_code}")
            return None

        soup = BeautifulSoup(response.text, 'html.parser')
        # Verificar CAPTCHA
        captcha_div = soup.find('div', {'id': 'captchacharacters'})
        recaptcha_div = soup.find('div', {'class': re.compile('g-recaptcha')})
        if 'captcha' in response.text.lower() or captcha_div or recaptcha_div:
            logging.info("CAPTCHA detectado en página de adición de dirección")
            print(f"**⚠️ CAPTCHA detectado en página de dirección. Intentando resolver...**")
            
            if recaptcha_div:
                site_key = recaptcha_div.get('data-sitekey')
                if site_key:
                    captcha_solution = solve_captcha(site_key, add_address_urls[domain])
                    if not captcha_solution:
                        print(f"**❌ No se pudo resolver el reCAPTCHA**")
                        return None
                    payload = {'g-recaptcha-response': captcha_solution}
                    response = session.post(
                        add_address_urls[domain],
                        headers={
                            'Host': f'www.{domain}',
                            'Content-Type': 'application/x-www-form-urlencoded',
                            'Origin': base_urls[domain],
                            'Referer': add_address_urls[domain],
                            'Accept': 'text/html,application/xhtml+xml,application/xml;q=0.9,image/webp,*/*;q=0.8',
                            'Accept-Language': 'en-US,en;q=0.9,es-MX;q=0.8,fr-FR;q=0.7,de-DE;q=0.7,it-IT;q=0.7,es-ES;q=0.7,ja-JP;q=0.7'
                        },
                        data=urlencode(payload),
                        timeout=15,
                        allow_redirects=True
                    )
                    soup = BeautifulSoup(response.text, 'html.parser')
                    logging.info(f"**Respuesta tras resolver reCAPTCHA (dirección)**: {response.status_code} - URL: {response.url}")
                else:
                    logging.error("No se encontró sitekey para reCAPTCHA")
                    print(f"**❌ No se encontró sitekey para reCAPTCHA**")
                    return None
            elif captcha_div:
                captcha_img = captcha_div.find('img')
                if captcha_img and captcha_img.get('src'):
                    captcha_solution = solve_captcha(None, add_address_urls[domain], is_image_captcha=True, image_url=captcha_img['src'])
                    if not captcha_solution:
                        print(f"**❌ No se pudo resolver el CAPTCHA de imagen**")
                        return None
                    payload = {'cvf_captcha_input': captcha_solution}
                    response = session.post(
                        add_address_urls[domain],
                        headers={
                            'Host': f'www.{domain}',
                            'Content-Type': 'application/x-www-form-urlencoded',
                            'Origin': base_urls[domain],
                            'Referer': add_address_urls[domain],
                            'Accept': 'text/html,application/xhtml+xml,application/xml;q=0.9,image/webp,*/*;q=0.8',
                            'Accept-Language': 'en-US,en;q=0.9,es-MX;q=0.8,fr-FR;q=0.7,de-DE;q=0.7,it-IT;q=0.7,es-ES;q=0.7,ja-JP;q=0.7'
                        },
                        data=urlencode(payload),
                        timeout=15,
                        allow_redirects=True
                    )
                    soup = BeautifulSoup(response.text, 'html.parser')
                    logging.info(f"**Respuesta tras resolver CAPTCHA de imagen (dirección)**: {response.status_code} - URL: {response.url}")
                else:
                    logging.error("No se encontró imagen de CAPTCHA")
                    print(f"**❌ No se encontró imagen de CAPTCHA**")
                    return None
            else:
                logging.error("CAPTCHA detectado pero no se encontró reCAPTCHA ni imagen")
                print(f"**❌ CAPTCHA detectado pero no se pudo identificar el tipo**")
                return None

        # Buscar formulario de dirección
        form = soup.find('form', {'id': 'address-ui-widgets-form'}) or soup.find('form', {'action': re.compile('add', re.I)})
        if not form:
            logging.error(f"No se encontró el formulario de dirección en {domain}")
            print(f"**❌ No se encontró el formulario de dirección en {domain}**")
            return None

        inputs = form.find_all('input', type='hidden')
        post_data = {inp.get('name'): inp.get('value') for inp in inputs if inp.get('name')}

        # Datos de la dirección por región
        address_data = {
            'amazon.ca': {
                'countryCode': 'CA',
                'fullName': 'Mark O. Montanez',
                'phone': f'1{random.randint(1000000000, 9999999999)}',
                'line1': '456 Bloor Street West',
                'city': 'Toronto',
                'state': 'ON',
                'postalCode': 'M5S 1X8'
            },
            'amazon.com.mx': {
                'countryCode': 'MX',
                'fullName': 'Juan Pérez',
                'phone': f'52{random.randint(1000000000, 9999999999)}',
                'line1': 'Calle Reforma 123',
                'city': 'Ciudad de México',
                'state': 'CDMX',
                'postalCode': '06000'
            },
            'amazon.com': {
                'countryCode': 'US',
                'fullName': 'John Doe',
                'phone': f'1{random.randint(1000000000, 9999999999)}',
                'line1': '123 Main Street',
                'city': 'New York',
                'state': 'NY',
                'postalCode': '10001'
            },
            'amazon.co.uk': {
                'countryCode': 'GB',
                'fullName': 'James Smith',
                'phone': f'44{random.randint(1000000000, 9999999999)}',
                'line1': '123 Oxford Street',
                'city': 'London',
                'state': '',
                'postalCode': 'W1D 1AA'
            },
            'amazon.de': {
                'countryCode': 'DE',
                'fullName': 'Hans Müller',
                'phone': f'49{random.randint(1000000000, 9999999999)}',
                'line1': 'Hauptstraße 12',
                'city': 'Berlin',
                'state': '',
                'postalCode': '10115'
            },
            'amazon.fr': {
                'countryCode': 'FR',
                'fullName': 'Pierre Dubois',
                'phone': f'33{random.randint(1000000000, 9999999999)}',
                'line1': '12 Rue de Rivoli',
                'city': 'Paris',
                'state': '',
                'postalCode': '75001'
            },
            'amazon.it': {
                'countryCode': 'IT',
                'fullName': 'Giuseppe Rossi',
                'phone': f'39{random.randint(1000000000, 9999999999)}',
                'line1': 'Via Roma 10',
                'city': 'Roma',
                'state': '',
                'postalCode': '00184'
            },
            'amazon.es': {
                'countryCode': 'ES',
                'fullName': 'Carlos García',
                'phone': f'34{random.randint(1000000000, 9999999999)}',
                'line1': 'Calle Mayor 15',
                'city': 'Madrid',
                'state': '',
                'postalCode': '28013'
            },
            'amazon.co.jp': {
                'countryCode': 'JP',
                'fullName': 'Taro Yamada',
                'phone': f'81{random.randint(1000000000, 9999999999)}',
                'line1': '1-2-3 Shibuya',
                'city': 'Tokyo',
                'state': '',
                'postalCode': '150-0002'
            },
            'amazon.com.au': {
                'countryCode': 'AU',
                'fullName': 'Emma Wilson',
                'phone': f'61{random.randint(1000000000, 9999999999)}',
                'line1': '123 George Street',
                'city': 'Sydney',
                'state': 'NSW',
                'postalCode': '2000'
            },
            'amazon.in': {
                'countryCode': 'IN',
                'fullName': 'Amit Sharma',
                'phone': f'91{random.randint(1000000000, 9999999999)}',
                'line1': '123 MG Road',
                'city': 'Mumbai',
                'state': 'Maharashtra',
                'postalCode': '400001'
            }
        }

        country_data = address_data[domain]
        post_data.update({
            "address-ui-widgets-countryCode": country_data['countryCode'],
            "address-ui-widgets-enterAddressFullName": country_data['fullName'],
            "address-ui-widgets-enterAddressPhoneNumber": country_data['phone'],
            "address-ui-widgets-enterAddressLine1": country_data['line1'],
            "address-ui-widgets-enterAddressLine2": "",
            "address-ui-widgets-enterAddressCity": country_data['city'],
            "address-ui-widgets-enterAddressStateOrRegion": country_data['state'],
            "address-ui-widgets-enterAddressPostalCode": country_data['postalCode'],
            "address-ui-widgets-use-as-my-default": "true",
            "address-ui-widgets-delivery-instructions-desktop-expander-context": '{"deliveryInstructionsDisplayMode" : "CDP_ONLY", "deliveryInstructionsClientName" : "YourAccountAddressBook", "deliveryInstructionsDeviceType" : "desktop", "deliveryInstructionsIsEditAddressFlow" : "false"}',
            "address-ui-widgets-addressFormButtonText": "save",
            "address-ui-widgets-addressFormHideHeading": "true",
            "address-ui-widgets-enableAddressDetails": "true",
            "address-ui-widgets-returnLegacyAddressID": "false",
            "address-ui-widgets-enableDeliveryInstructions": "true",
            "address-ui-widgets-enableAddressWizardInlineSuggestions": "false",
            "address-ui-widgets-enableEmailAddress": "false",
            "address-ui-widgets-enableAddressTips": "true",
            "address-ui-widgets-clientName": "YourAccountAddressBook",
            "address-ui-widgets-enableAddressWizardForm": "true",
            "address-ui-widgets-delivery-instructions-data": f'{{"initialCountryCode":"{country_data["countryCode"]}"}}',
            "address-ui-widgets-enableLatestAddressWizardForm": "true",
            "address-ui-widgets-avsSuppressSoftblock": "false",
            "address-ui-widgets-avsSuppressSuggestion": "false"
        })

        post_url = urljoin(base_urls[domain], form.get('action') or "/a/addresses/add")
        headers.update({
            "Content-Type": "application/x-www-form-urlencoded",
            "Origin": base_urls[domain],
            "Referer": add_address_urls[domain]
        })

        # Enviar la dirección
        response = session.post(
            post_url,
            data=urlencode(post_data),
            headers=headers,
            timeout=15,
            allow_redirects=True
        )
        logging.info(f"**Respuesta envío de dirección ({domain})**: {response.status_code} - URL: {response.url}")
        with open(f'address_response_{email}_{domain}.html', 'w', encoding='utf-8') as f:
            f.write(response.text)

        soup = BeautifulSoup(response.text, 'html.parser')
        error_div = soup.find('div', {'class': re.compile('a-alert-error|a-alert-warning|a-box-error', re.I)})
        if error_div:
            error_message = error_div.get_text(strip=True)
            logging.error(f"Error al enviar dirección: {error_message}")
            print(f"**❌ Error al enviar dirección para {email} ({domain})**: {error_message}")
            return None

        if response.status_code != 200 or 'address' in response.url.lower():
            logging.error(f"Fallo al enviar dirección para {email} en {domain}. Status: {response.status_code}, URL: {response.url}")
            print(f"**❌ Fallo al enviar dirección para {email} ({domain})**: {response.status_code}")
            return None

        # Verificar la libreta de direcciones
        response = session.get(address_book_urls[domain], headers=headers, timeout=15, allow_redirects=True)
        logging.info(f"**Respuesta verificación de libreta de direcciones ({domain})**: {response.status_code} - URL: {response.url}")
        with open(f'address_book_verify_{email}_{domain}.html', 'w', encoding='utf-8') as f:
            f.write(response.text)

        soup = BeautifulSoup(response.text, 'html.parser')
        address_div = soup.find('div', string=re.compile(country_data['line1'], re.I))
        if not address_div:
            logging.error(f"No se encontró la dirección en la libreta de direcciones para {email} ({domain})")
            print(f"**❌ No se encontró la dirección en la libreta de direcciones para {email} ({domain})**")
            return None

        # Acceder a la página de billetera para generar cookies
        response = session.get(wallet_urls[domain], headers=headers, timeout=15, allow_redirects=True)
        logging.info(f"**Respuesta página de billetera ({domain})**: {response.status_code} - URL: {response.url}")
        with open(f'wallet_page_{email}_{domain}.html', 'w', encoding='utf-8') as f:
            f.write(response.text)

        if response.status_code != 200:
            logging.error(f"Fallo al acceder a la página de billetera para {email} en {domain}: {response.status_code}")
            print(f"**❌ Fallo al acceder a la página de billetera para {email} ({domain})**: {response.status_code}")
            return None

        # Extraer cookies
        cookies = session.cookies.get_dict()
        cookie_str = '; '.join([f"{key}={value}" for key, value in cookies.items()])
        logging.info(f"Cookies generadas en billetera para {email} en {domain}: {cookie_str}")
        print(f"**🍪 Cookies generadas para {email} ({domain})**:\n{cookie_str}\n🟢")
        return cookie_str

    except Exception as e:
        logging.error(f"Error al agregar dirección para {email} en {domain}: {str(e)}")
        print(f"**❌ Error al agregar dirección para {email} ({domain})**: {str(e)}")
        return None

# Función para crear cuenta de Amazon y obtener cookies
async def create_amazon_account(domain, email=None, token=None, service=None):
    playwright = None
    browser = None
    page = None
    
    try:
        # Inicializar Playwright con mejor configuración
        playwright = await async_playwright().start()
        browser = await playwright.chromium.launch(
            headless=True,
            args=[
                '--no-sandbox',
                '--disable-setuid-sandbox',
                '--disable-dev-shm-usage',
                '--disable-accelerated-2d-canvas',
                '--no-first-run',
                '--no-zygote',
                '--disable-gpu'
            ]
        )
        context = await browser.new_context(
            viewport={'width': 1280, 'height': 720},
            user_agent=random.choice(USER_AGENTS)
        )
        page = await context.new_page()
        
        # Configurar headers
        user_agent = random.choice(USER_AGENTS)
        await page.set_extra_http_headers({
            'User-Agent': user_agent,
            'Accept': 'text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8',
            'Accept-Language': 'en-US,en;q=0.9',
        })
        
        session = requests.Session()
        for key, value in INITIAL_COOKIES[domain].items():
            session.cookies.set(key, value, domain=f".{domain}")
        session.headers.update({'User-Agent': user_agent})
        session.proxies = {'http': f'http://{proxy_auth}@{proxy}', 'https': f'http://{proxy_auth}@{proxy}'}

        success, proxy_result = test_proxy(session)
        if not success:
            print(f"**❌ Error al probar el proxy**: {proxy_result}")
            return None
        print(f"**🛡️ Proxy funcionando** 🟢")

        for attempt in range(3):
            try:
                if not email:
                    email, token, service = await generate_temp_email()
                    if not email:
                        return None

                password = f"Pass{random.randint(1000, 9999)}{uuid.uuid4().hex[:8]}"
                first_name = ''.join(random.choices('abcdefghijklmnopqrstuvwxyz', k=5)).capitalize()
                last_name = ''.join(random.choices('abcdefghijklmnopqrstuvwxyz', k=5)).capitalize()
                fullname = f"{first_name} {last_name}"

                manage_urls = {
                    'amazon.ca': 'https://www.amazon.ca/ap/register?openid.mode=checkid_setup&openid.ns=http%3A%2F%2Fspecs.openid.net%2Fauth%2F2.0&openid.identity=http%3A%2F%2Fspecs.openid.net%2Fauth%2F2.0%2Fidentifier_select&openid.claimed_id=http%3A%2F%2Fspecs.openid.net%2Fauth%2F2.0%2Fidentifier_select&openid.ns.pape=http%3A%2F%2Fspecs.openid.net%2Fextensions%2Fpape%2F1.0&showRememberMe=true&openid.pape.max_auth_age=3600&pageId=anywhere_ca&prepopulatedLoginId=&openid.assoc_handle=anywhere_v2_ca&openid.return_to=https%3A%2F%2Fwww.amazon.ca%2Fyour-account&policy_handle=Retail-Checkout',
                    'amazon.com.mx': 'https://www.amazon.com.mx/ap/register?openid.mode=checkid_setup&openid.ns=http%3A%2F%2Fspecs.openid.net%2Fauth%2F2.0&openid.identity=http%3A%2F%2Fspecs.openid.net%2Fauth%2F2.0%2Fidentifier_select&openid.claimed_id=http%3A%2F%2Fspecs.openid.net%2Fauth%2F2.0%2Fidentifier_select&openid.ns.pape=http%3A%2F%2Fspecs.openid.net%2Fextensions%2Fpape%2F1.0&showRememberMe=true&openid.pape.max_auth_age=3600&pageId=anywhere_mx&prepopulatedLoginId=&openid.assoc_handle=anywhere_v2_mx&openid.return_to=https%3A%2F%2Fwww.amazon.com.mx%2Fyour-account&policy_handle=Retail-Checkout',
                    'amazon.com': 'https://www.amazon.com/ap/register?openid.mode=checkid_setup&openid.ns=http%3A%2F%2Fspecs.openid.net%2Fauth%2F2.0&openid.identity=http%3A%2F%2Fspecs.openid.net%2Fauth%2F2.0%2Fidentifier_select&openid.claimed_id=http%3A%2F%2Fspecs.openid.net%2Fauth%2F2.0%2Fidentifier_select&openid.ns.pape=http%3A%2F%2Fspecs.openid.net%2Fextensions%2Fpape%2F1.0&showRememberMe=true&openid.pape.max_auth_age=3600&pageId=anywhere_us&prepopulatedLoginId=&openid.assoc_handle=anywhere_v2_us&openid.return_to=https%3A%2F%2Fwww.amazon.com%2Fyour-account&policy_handle=Retail-Checkout',
                    'amazon.co.uk': 'https://www.amazon.co.uk/ap/register?openid.mode=checkid_setup&openid.ns=http%3A%2F%2Fspecs.openid.net%2Fauth%2F2.0&openid.identity=http%3A%2F%2Fspecs.openid.net%2Fauth%2F2.0%2Fidentifier_select&openid.claimed_id=http%3A%2F%2Fspecs.openid.net%2Fauth%2F2.0%2Fidentifier_select&openid.ns.pape=http%3A%2F%2Fspecs.openid.net%2Fextensions%2Fpape%2F1.0&showRememberMe=true&openid.pape.max_auth_age=3600&pageId=anywhere_uk&prepopulatedLoginId=&openid.assoc_handle=anywhere_v2_uk&openid.return_to=https%3A%2F%2Fwww.amazon.co.uk%2Fyour-account&policy_handle=Retail-Checkout',
                    'amazon.de': 'https://www.amazon.de/ap/register?openid.mode=checkid_setup&openid.ns=http%3A%2F%2Fspecs.openid.net%2Fauth%2F2.0&openid.identity=http%3A%2F%2Fspecs.openid.net%2Fauth%2F2.0%2Fidentifier_select&openid.claimed_id=http%3A%2F%2Fspecs.openid.net%2Fauth%2F2.0%2Fidentifier_select&openid.ns.pape=http%3A%2F%2Fspecs.openid.net%2Fextensions%2Fpape%2F1.0&showRememberMe=true&openid.pape.max_auth_age=3600&pageId=anywhere_de&prepopulatedLoginId=&openid.assoc_handle=anywhere_v2_de&openid.return_to=https%3A%2F%2Fwww.amazon.de%2Fyour-account&policy_handle=Retail-Checkout',
                    'amazon.fr': 'https://www.amazon.fr/ap/register?openid.mode=checkid_setup&openid.ns=http%3A%2F%2Fspecs.openid.net%2Fauth%2F2.0&openid.identity=http%3A%2F%2Fspecs.openid.net%2Fauth%2F2.0%2Fidentifier_select&openid.claimed_id=http%3A%2F%2Fspecs.openid.net%2Fauth%2F2.0%2Fidentifier_select&openid.ns.pape=http%3A%2F%2Fspecs.openid.net%2Fextensions%2Fpape%2F1.0&showRememberMe=true&openid.pape.max_auth_age=3600&pageId=anywhere_fr&prepopulatedLoginId=&openid.assoc_handle=anywhere_v2_fr&openid.return_to=https%3A%2F%2Fwww.amazon.fr%2Fyour-account&policy_handle=Retail-Checkout',
                    'amazon.it': 'https://www.amazon.it/ap/register?openid.mode=checkid_setup&openid.ns=http%3A%2F%2Fspecs.openid.net%2Fauth%2F2.0&openid.identity=http%3A%2F%2Fspecs.openid.net%2Fauth%2F2.0%2Fidentifier_select&openid.claimed_id=http%3A%2F%2Fspecs.openid.net%2Fauth%2F2.0%2Fidentifier_select&openid.ns.pape=http%3A%2F%2Fspecs.openid.net%2Fextensions%2Fpape%2F1.0&showRememberMe=true&openid.pape.max_auth_age=3600&pageId=anywhere_it&prepopulatedLoginId=&openid.assoc_handle=anywhere_v2_it&openid.return_to=https%3A%2F%2Fwww.amazon.it%2Fyour-account&policy_handle=Retail-Checkout',
                    'amazon.es': 'https://www.amazon.es/ap/register?openid.mode=checkid_setup&openid.ns=http%3A%2F%2Fspecs.openid.net%2Fauth%2F2.0&openid.identity=http%3A%2F%2Fspecs.openid.net%2Fauth%2F2.0%2Fidentifier_select&openid.claimed_id=http%3A%2F%2Fspecs.openid.net%2Fauth%2F2.0%2Fidentifier_select&openid.ns.pape=http%3A%2F%2Fspecs.openid.net%2Fauth%2F2.0%2Fidentifier_select&openid.ns.pape=http%3A%2F%2Fspecs.openid.net%2Fextensions%2Fpape%2F1.0&showRememberMe=true&openid.pape.max_auth_age=3600&pageId=anywhere_es&prepopulatedLoginId=&openid.assoc_handle=anywhere_v2_es&openid.return_to=https%3A%2F%2Fwww.amazon.es%2Fyour-account&policy_handle=Retail-Checkout',
                    'amazon.co.jp': 'https://www.amazon.co.jp/ap/register?openid.mode=checkid_setup&openid.ns=http%3A%2F%2Fspecs.openid.net%2Fauth%2F2.0&openid.identity=http%3A%2F%2Fspecs.openid.net%2Fauth%2F2.0%2Fidentifier_select&openid.claimed_id=http%3A%2F%2Fspecs.openid.net%2Fauth%2F2.0%2Fidentifier_select&openid.ns.pape=http%3A%2F%2Fspecs.openid.net%2Fextensions%2Fpape%2F1.0&showRememberMe=true&openid.pape.max_auth_age=3600&pageId=anywhere_jp&prepopulatedLoginId=&openid.assoc_handle=anywhere_v2_jp&openid.return_to=https%3A%2F%2Fwww.amazon.co.jp%2Fyour-account&policy_handle=Retail-Checkout',
                    'amazon.com.au': 'https://www.amazon.com.au/ap/register?openid.mode=checkid_setup&openid.ns=http%3A%2F%2Fspecs.openid.net%2Fauth%2F2.0&openid.identity=http%3A%2F%2Fspecs.openid.net%2Fauth%2F2.0%2Fidentifier_select&openid.claimed_id=http%3A%2F%2Fspecs.openid.net%2Fauth%2F2.0%2Fidentifier_select&openid.ns.pape=http%3A%2F%2Fspecs.openid.net%2Fextensions%2Fpape%2F1.0&showRememberMe=true&openid.pape.max_auth_age=3600&pageId=anywhere_au&prepopulatedLoginId=&openid.assoc_handle=anywhere_v2_au&openid.return_to=https%3A%2F%2Fwww.amazon.com.au%2Fyour-account&policy_handle=Retail-Checkout',
                    'amazon.in': 'https://www.amazon.in/ap/register?openid.mode=checkid_setup&openid.ns=http%3A%2F%2Fspecs.openid.net%2Fauth%2F2.0&openid.identity=http%3A%2F%2Fspecs.openid.net%2Fauth%2F2.0%2Fidentifier_select&openid.claimed_id=http%3A%2F%2Fspecs.openid.net%2Fauth%2F2.0%2Fidentifier_select&openid.ns.pape=http%3A%2F%2Fspecs.openid.net%2Fextensions%2Fpape%2F1.0&showRememberMe=true&openid.pape.max_auth_age=3600&pageId=anywhere_in&prepopulatedLoginId=&openid.assoc_handle=anywhere_v2_in&openid.return_to=https%3A%2F%2Fwww.amazon.in%2Fyour-account&policy_handle=Retail-Checkout'
                }

                # Cargar página con JavaScript - aumentar tiempo de espera y mejor manejo de errores
                print(f"**🌐 Cargando página de registro...**")
                
                # Intentar cargar la página con reintentos
                page_loaded = False
                for load_attempt in range(3):
                    try:
                        await page.goto(manage_urls[domain], wait_until='networkidle', timeout=90000)  # Increased timeout to 90 seconds
                        await page.wait_for_timeout(5000)  # Esperar más tiempo para que JS se cargue
                        page_loaded = True
                        break
                    except Exception as load_error:
                        print(f"**⚠️ Intento {load_attempt + 1} falló al cargar página: {str(load_error)}**")
                        if load_attempt < 2:
                            await asyncio.sleep(2 ** load_attempt)
                        else:
                            raise load_error
                
                if not page_loaded:
                    print(f"**❌ No se pudo cargar la página de registro después de 3 intentos**")
                    await asyncio.sleep(2 ** attempt)
                    continue
                
                # Intentar esperar por elementos específicos del formulario
                try:
                    await page.wait_for_selector('input[name="customerName"], input[name="email"], form[id*="register"], form[action*="register"]', timeout=10000)
                except:
                    print(f"**⚠️ No se encontraron elementos del formulario inmediatamente, continuando...**")
                
                content = await page.content()
                soup = BeautifulSoup(content, 'html.parser')

                with open(f'register_page_{email}_{domain}_attempt_{attempt+1}.html', 'w', encoding='utf-8') as f:
                    f.write(content)

                # Mejorar la detección del formulario
                form = None
                
                # Buscar por diferentes criterios
                form_selectors = [
                    ('form', {'id': 'ap_register_form'}),
                    ('form', {'name': 'register'}),
                    ('form', {'action': re.compile('register', re.I)}),
                    ('form', {'method': 'post'}),
                    ('form', {'id': re.compile('register', re.I)}),
                    ('form', {'class': re.compile('register', re.I)})
                ]
                
                for selector in form_selectors:
                    form = soup.find(*selector)
                    if form:
                        print(f"**✅ Formulario encontrado con selector: {selector}**")
                        break
                
                # Si no se encuentra formulario, buscar cualquier form que contenga campos de registro
                if not form:
                    all_forms = soup.find_all('form')
                    for f in all_forms:
                        if f.find('input', {'name': 'email'}) or f.find('input', {'name': 'customerName'}) or f.find('input', {'type': 'password'}):
                            form = f
                            print(f"**✅ Formulario encontrado por campos de entrada**")
                            break
                
                if not form:
                    print(f"**❌ No se encontró el formulario de registro en {domain}**")
                    await asyncio.sleep(2 ** attempt)
                    continue

                # Usar Playwright para llenar y enviar el formulario
                print(f"**📝 Llenando formulario de registro...**")
                
                try:
                    # Esperar más tiempo y usar selectores más flexibles
                    await page.wait_for_timeout(3000)  # Esperar adicional
                    
                    # Intentar diferentes selectores para el nombre
                    name_selectors = [
                        'input[name="customerName"]',
                        'input[name="name"]',
                        'input[placeholder*="name" i]',
                        'input[id*="name" i]',
                        '#ap_customer_name',
                        'input[type="text"]:first-of-type'
                    ]
                    
                    name_filled = False
                    for selector in name_selectors:
                        try:
                            await page.wait_for_selector(selector, timeout=5000)
                            await page.fill(selector, fullname)
                            print(f"**✅ Nombre llenado con selector: {selector}**")
                            name_filled = True
                            break
                        except:
                            continue
                    
                    if not name_filled:
                        print("**⚠️ No se pudo encontrar campo de nombre, intentando continuar...**")
                    
                    # Email
                    email_selectors = [
                        'input[name="email"]',
                        'input[type="email"]',
                        'input[placeholder*="email" i]',
                        '#ap_email'
                    ]
                    
                    for selector in email_selectors:
                        try:
                            await page.wait_for_selector(selector, timeout=5000)
                            await page.fill(selector, email)
                            print(f"**✅ Email llenado con selector: {selector}**")
                            break
                        except:
                            continue
                    
                    # Password
                    password_selectors = [
                        'input[name="password"]',
                        'input[type="password"]:first-of-type',
                        'input[placeholder*="password" i]',
                        '#ap_password'
                    ]
                    
                    for selector in password_selectors:
                        try:
                            await page.wait_for_selector(selector, timeout=5000)
                            await page.fill(selector, password)
                            print(f"**✅ Password llenado con selector: {selector}**")
                            break
                        except:
                            continue
                    
                    # Password confirmation
                    confirm_selectors = [
                        'input[name="passwordCheck"]',
                        'input[name="password_check"]',
                        'input[type="password"]:nth-of-type(2)',
                        'input[placeholder*="confirm" i]',
                        '#ap_password_check'
                    ]
                    
                    for selector in confirm_selectors:
                        try:
                            await page.wait_for_selector(selector, timeout=5000)
                            await page.fill(selector, password)
                            print(f"**✅ Confirmación de password llenada con selector: {selector}**")
                            break
                        except:
                            continue
                    
                    # Phone number (required for amazon.com)
                    if domain == 'amazon.com':
                        phone_selectors = [
                            'input[name="phoneNumber"]',
                            'input[name="mobileNumber"]',
                            'input[type="tel"]',
                            'input[placeholder*="phone" i]',
                            'input[placeholder*="mobile" i]',
                            '#ap_phone_number'
                        ]
                        
                        phone_number = f"+1{random.randint(200, 999)}{random.randint(1000000, 9999999)}"  # Generate US phone number
                        
                        for selector in phone_selectors:
                            try:
                                await page.wait_for_selector(selector, timeout=5000)
                                await page.fill(selector, phone_number)
                                print(f"**✅ Número de teléfono llenado con selector: {selector}**")
                                break
                            except:
                                continue
                    
                    # Buscar y hacer clic en checkbox de términos si existe
                    try:
                        await page.check('input[name="agreement"]', timeout=5000)
                        print("**✅ Checkbox de términos marcado**")
                    except:
                        try:
                            await page.check('input[type="checkbox"]', timeout=5000)
                            print("**✅ Checkbox encontrado y marcado**")
                        except:
                            print("**⚠️ No se encontró checkbox de términos**")
                    
                    # Hacer clic en el botón de registro
                    submit_selectors = [
                        'input[type="submit"]',
                        'button[type="submit"]',
                        '#continue',
                        '.a-button-input',
                        'button:contains("Create")',
                        'input[value*="Create" i]',
                        'input[value*="Register" i]'
                    ]
                    
                    submit_clicked = False
                    for selector in submit_selectors:
                        try:
                            await page.wait_for_selector(selector, timeout=5000)
                            await page.click(selector)
                            print(f"**✅ Botón submit clickeado con selector: {selector}**")
                            submit_clicked = True
                            break
                        except:
                            continue
                    
                    if not submit_clicked:
                        print("**❌ No se pudo encontrar botón de submit**")
                        await asyncio.sleep(2 ** attempt)
                        continue
                    
                    # Esperar a que se complete el envío
                    await page.wait_for_load_state('networkidle', timeout=30000)
                    
                    # Verificar si el registro fue exitoso
                    current_url = page.url
                    content = await page.content()
                    
                    with open(f'register_response_{email}_{domain}_attempt_{attempt+1}.html', 'w', encoding='utf-8') as f:
                        f.write(content)
                    
                    soup = BeautifulSoup(content, 'html.parser')
                    
                    # Verificar errores
                    error_div = soup.find('div', {'class': re.compile('a-alert-error|a-alert-warning|a-box-error', re.I)})
                    if error_div:
                        error_message = error_div.get_text(strip=True)
                        print(f"**❌ Error al registrar cuenta para {email} ({domain})**: {error_message}")
                        await asyncio.sleep(2 ** attempt)
                        continue
                    
                    # Verificar si estamos en la página de cuenta (éxito)
                    if 'your-account' in current_url.lower() or 'account' in current_url.lower() or 'homepage' in current_url.lower():
                        print(f"**✅ Cuenta creada: {email}** 🟢\n**Contraseña**: {password}")
                        
                        # Agregar dirección antes de obtener cookies
                        print(f"**📍 Agregando dirección para {domain}...**")
                        address_added = await add_address_playwright(page, domain, fullname)
                        if address_added:
                            print(f"**✅ Dirección agregada exitosamente**")
                        else:
                            print(f"**⚠️ No se pudo agregar dirección, continuando...**")
                        
                        # Navegar a la página de wallet para obtener cookies completas
                        print(f"**💳 Navegando a página de wallet para cookies completas...**")
                        try:
                            await page.goto(wallet_urls[domain], wait_until='networkidle', timeout=30000)
                            await page.wait_for_timeout(3000)  # Esperar a que cargue
                            print(f"**✅ Página de wallet cargada**")
                        except Exception as wallet_error:
                            print(f"**⚠️ Error al cargar página de wallet: {str(wallet_error)}**")
                        
                        # Obtener cookies finales de la sesión del navegador
                        cookies = await context.cookies()
                        cookie_dict = {cookie['name']: cookie['value'] for cookie in cookies}
                        
                        print(f"**🍪 Cookies obtenidas exitosamente para {domain}**")
                        return cookie_dict
                    
                    print(f"**❌ Fallo al registrar cuenta para {email} ({domain}) - URL actual: {current_url}**")
                    await asyncio.sleep(2 ** attempt)
                    continue
                    
                except Exception as form_error:
                    print(f"**❌ Error al llenar/enviar formulario para {email} ({domain})**: {str(form_error)}")
                    await asyncio.sleep(2 ** attempt)
                    continue
                    
                    with open(f'register_response_{email}_{domain}_attempt_{attempt+1}.html', 'w', encoding='utf-8') as f:
                        f.write(content)
                    
                    soup = BeautifulSoup(content, 'html.parser')
                    
                    # Verificar errores
                    error_div = soup.find('div', {'class': re.compile('a-alert-error|a-alert-warning|a-box-error', re.I)})
                    if error_div:
                        error_message = error_div.get_text(strip=True)
                        print(f"**❌ Error al registrar cuenta para {email} ({domain})**: {error_message}")
                        await asyncio.sleep(2 ** attempt)
                        continue
                    
                    # Verificar si estamos en la página de cuenta (éxito)
                    if 'your-account' in current_url.lower() or 'account' in current_url.lower():
                        print(f"**✅ Cuenta creada: {email}** 🟢\n**Contraseña**: {password}")
                        # Obtener cookies de la sesión del navegador
                        cookies = await page.context.cookies()
                        cookie_dict = {cookie['name']: cookie['value'] for cookie in cookies}
                        
                        print(f"**🍪 Cookies obtenidas exitosamente para {domain}**")
                        return cookie_dict
                    
                    print(f"**❌ Fallo al registrar cuenta para {email} ({domain}) - URL actual: {current_url}**")
                    await asyncio.sleep(2 ** attempt)
                    continue
                    
                except Exception as form_error:
                    print(f"**❌ Error al llenar/enviar formulario para {email} ({domain})**: {str(form_error)}")
                    await asyncio.sleep(2 ** attempt)
                    continue

            except Exception as e:
                print(f"**❌ Error al crear cuenta para {email} ({domain})**: {str(e)}")
                await asyncio.sleep(2 ** attempt)
                continue

        print(f"**❌ No se pudo crear cuenta para {email} ({domain}) después de 3 intentos**")
        return None
        
    finally:
        if page:
            await page.close()
        if 'context' in locals():
            await context.close()
        if browser:
            await browser.close()
        if playwright:
            await playwright.stop()
# Función para agregar dirección usando Playwright
async def add_address_playwright(page, domain, fullname):
    try:
        # Para Amazon, la dirección se agrega a través del sistema de pagos/wallet
        # Navegar a la página de manage one-click settings
        manage_oneclick_url = f"https://www.{domain}/cpe/yourpayments/settings/manageoneclick?ref_=ya_d_l_change_1_click&claim_type=MobileNumber&new_account=1&"
        await page.goto(manage_oneclick_url, wait_until='networkidle', timeout=30000)
        await page.wait_for_timeout(3000)
        
        # Esperar a que se cargue el widget de pagos
        try:
            await page.wait_for_selector('.a-box, .payment-widget, [data-widget-type]', timeout=10000)
        except:
            print("**⚠️ Widget de pagos no encontrado, intentando continuar...**")
        
        # Generar datos de dirección según el dominio
        if domain == 'amazon.com':
            address_data = {
                'full_name': fullname,
                'address_line1': f"{random.randint(100, 999)} Main St",
                'city': 'New York',
                'state': 'NY',
                'zip_code': f"{random.randint(10000, 99999)}",
                'phone': f"+1{random.randint(200, 999)}{random.randint(1000000, 9999999)}"
            }
        elif domain == 'amazon.ca':
            address_data = {
                'full_name': fullname,
                'address_line1': f"{random.randint(100, 999)} Queen St",
                'city': 'Toronto',
                'province': 'ON',
                'postal_code': f"{random.choice('ABCDEFGHIJKLMNOPQRSTUVWXYZ')}{random.randint(1,9)}{random.choice('ABCDEFGHIJKLMNOPQRSTUVWXYZ')} {random.randint(1,9)}{random.choice('ABCDEFGHIJKLMNOPQRSTUVWXYZ')}{random.randint(1,9)}",
                'phone': f"+1{random.randint(200, 999)}{random.randint(1000000, 9999999)}"
            }
        elif domain == 'amazon.co.uk':
            address_data = {
                'full_name': fullname,
                'address_line1': f"{random.randint(1, 999)} High Street",
                'city': 'London',
                'postcode': f"{random.choice('ABCDEFGHIJKLMNOPQRSTUVWXYZ')}{random.choice('ABCDEFGHIJKLMNOPQRSTUVWXYZ')}{random.randint(1,99)} {random.randint(1,9)}{random.choice('ABCDEFGHIJKLMNOPQRSTUVWXYZ')}{random.choice('ABCDEFGHIJKLMNOPQRSTUVWXYZ')}",
                'phone': f"+44{random.randint(7000000000, 7999999999)}"
            }
        elif domain == 'amazon.de':
            address_data = {
                'full_name': fullname,
                'address_line1': f"{random.randint(1, 999)} Hauptstraße",
                'city': 'Berlin',
                'postal_code': f"{random.randint(10000, 99999)}",
                'phone': f"+49{random.randint(1500000000, 1999999999)}"
            }
        elif domain == 'amazon.fr':
            address_data = {
                'full_name': fullname,
                'address_line1': f"{random.randint(1, 999)} Rue de la Paix",
                'city': 'Paris',
                'postal_code': f"{random.randint(75000, 75999)}",
                'phone': f"+33{random.randint(600000000, 799999999)}"
            }
        elif domain == 'amazon.it':
            address_data = {
                'full_name': fullname,
                'address_line1': f"Via Roma {random.randint(1, 999)}",
                'city': 'Rome',
                'postal_code': f"{random.randint(1000, 99999)}",
                'phone': f"+39{random.randint(3000000000, 3999999999)}"
            }
        elif domain == 'amazon.es':
            address_data = {
                'full_name': fullname,
                'address_line1': f"Calle Mayor {random.randint(1, 999)}",
                'city': 'Madrid',
                'postal_code': f"{random.randint(28000, 28999)}",
                'phone': f"+34{random.randint(600000000, 799999999)}"
            }
        elif domain == 'amazon.com.mx':
            address_data = {
                'full_name': fullname,
                'address_line1': f"Calle {random.randint(1, 999)} Colonia Centro",
                'city': 'Mexico City',
                'state': 'CDMX',
                'postal_code': f"{random.randint(1000, 99999)}",
                'phone': f"+52{random.randint(5500000000, 5599999999)}"
            }
        elif domain == 'amazon.co.jp':
            address_data = {
                'full_name': fullname,
                'address_line1': f"{random.randint(1, 999)} Chome, Shibuya",
                'city': 'Tokyo',
                'prefecture': 'Tokyo',
                'postal_code': f"{random.randint(1000000, 9999999)}",
                'phone': f"+81{random.randint(80000000000, 99999999999)}"
            }
        elif domain == 'amazon.com.au':
            address_data = {
                'full_name': fullname,
                'address_line1': f"{random.randint(1, 999)} George Street",
                'city': 'Sydney',
                'state': 'NSW',
                'postal_code': f"{random.randint(2000, 2999)}",
                'phone': f"+61{random.randint(400000000, 499999999)}"
            }
        elif domain == 'amazon.in':
            address_data = {
                'full_name': fullname,
                'address_line1': f"{random.randint(1, 999)} MG Road",
                'city': 'Mumbai',
                'state': 'Maharashtra',
                'postal_code': f"{random.randint(400001, 400099)}",
                'phone': f"+91{random.randint(9000000000, 9999999999)}"
            }
        else:
            return False
        
        # Intentar interactuar con el widget de pagos para agregar dirección
        # Buscar botones o enlaces para agregar dirección
        add_address_selectors = [
            'a:contains("Add address")',
            'button:contains("Add")',
            '.a-button:contains("Add")',
            '[data-action="add-address"]',
            '.add-address-button'
        ]
        
        address_added = False
        for selector in add_address_selectors:
            try:
                await page.wait_for_selector(selector, timeout=5000)
                await page.click(selector)
                await page.wait_for_timeout(2000)
                address_added = True
                break
            except:
                continue
        
        if not address_added:
            # Si no hay botón específico, intentar llenar campos directamente si están visibles
            print("**⚠️ Intentando llenar campos de dirección directamente...**")
            
            # Nombre completo
            name_selectors = ['input[name*="name"]', 'input[placeholder*="name" i]', '#address-ui-widgets-enterAddressFullName']
            for selector in name_selectors:
                try:
                    await page.wait_for_selector(selector, timeout=3000)
                    await page.fill(selector, address_data['full_name'])
                    break
                except:
                    continue
            
            # Dirección
            address_selectors = ['input[name*="address"]', 'input[placeholder*="address" i]', '#address-ui-widgets-enterAddressLine1']
            for selector in address_selectors:
                try:
                    await page.wait_for_selector(selector, timeout=3000)
                    await page.fill(selector, address_data['address_line1'])
                    break
                except:
                    continue
            
            # Ciudad
            city_selectors = ['input[name*="city"]', 'input[placeholder*="city" i]', '#address-ui-widgets-enterAddressCity']
            for selector in city_selectors:
                try:
                    await page.wait_for_selector(selector, timeout=3000)
                    await page.fill(selector, address_data['city'])
                    break
                except:
                    continue
            
            # Código postal/ZIP
            postal_selectors = ['input[name*="postal"]', 'input[name*="zip"]', 'input[placeholder*="postal" i]', '#address-ui-widgets-enterAddressPostalCode']
            postal_key = 'zip_code' if 'zip_code' in address_data else ('postal_code' if 'postal_code' in address_data else 'postcode')
            for selector in postal_selectors:
                try:
                    await page.wait_for_selector(selector, timeout=3000)
                    await page.fill(selector, address_data[postal_key])
                    break
                except:
                    continue
            
            # Teléfono
            phone_selectors = ['input[name*="phone"]', 'input[type="tel"]', '#address-ui-widgets-enterAddressPhoneNumber']
            for selector in phone_selectors:
                try:
                    await page.wait_for_selector(selector, timeout=3000)
                    await page.fill(selector, address_data['phone'])
                    break
                except:
                    continue
            
            # Intentar guardar
            save_selectors = ['button[type="submit"]', '.a-button-input', 'input[value*="Save" i]', 'button:contains("Save")']
            for selector in save_selectors:
                try:
                    await page.wait_for_selector(selector, timeout=3000)
                    await page.click(selector)
                    await page.wait_for_load_state('networkidle', timeout=10000)
                    address_added = True
                    break
                except:
                    continue
        
        # Verificar si la dirección fue agregada exitosamente
        if address_added:
            await page.wait_for_timeout(2000)  # Esperar a que se procese
            return True
        
        return False
        
    except Exception as e:
        print(f"**❌ Error al agregar dirección: {str(e)}**")
        return False

# Función para convertir cookies al formato esperado
def convert_cookie(cookies_dict, country_code):
    """Convierte un diccionario de cookies a formato de string legible"""
    if not cookies_dict or not isinstance(cookies_dict, dict):
        return "No se pudieron obtener cookies"
    
    # Formatear cookies como string
    cookie_lines = []
    for name, value in cookies_dict.items():
        cookie_lines.append(f"{name}={value}")
    
    return "; ".join(cookie_lines)

# Comando /start (ahora función de terminal)
def start():
    print("**¡Bot iniciado!** Usa la opción de menú para generar cookies.")

# Comando /cookies (ahora función de terminal)
async def cookies(country=None):
    if country:
        country = country.upper()
        country_to_domain = {
            'CA': 'amazon.ca',
            'MX': 'amazon.com.mx',
            'US': 'amazon.com',
            'UK': 'amazon.co.uk',
            'DE': 'amazon.de',
            'FR': 'amazon.fr',
            'IT': 'amazon.it',
            'ES': 'amazon.es',
            'JP': 'amazon.co.jp',
            'AU': 'amazon.com.au',
            'IN': 'amazon.in'
        }
        if country in country_to_domain:
            domain = country_to_domain[country]
            print(f"**🔄 Generando cookies para {domain}...**")
            cookies_result = await create_amazon_account(domain)
            if cookies_result:
                converted_cookies = convert_cookie(cookies_result, country)
                print(f"**🍪 Cookies generadas para {domain}**:\n{converted_cookies}\n🟢")
            else:
                print(f"**❌ No se pudieron generar cookies para {domain}**")
        else:
            print(f"**❌ País no soportado: {country}. Usa CA, MX, US, UK, DE, FR, IT, ES, JP, AU, IN**")
    else:
        print("**🔄 Generando cookies para todos los dominios...**")
        for domain in domains:
            print(f"**🔄 Generando cookies para {domain}...**")
            cookies_result = await create_amazon_account(domain)
            if cookies_result:
                country_code = domain.split('.')[-1].upper()
                if domain == 'amazon.com':
                    country_code = 'US'
                elif domain == 'amazon.com.mx':
                    country_code = 'MX'
                elif domain == 'amazon.com.au':
                    country_code = 'AU'
                elif domain == 'amazon.co.uk':
                    country_code = 'UK'
                elif domain == 'amazon.co.jp':
                    country_code = 'JP'
                converted_cookies = convert_cookie(cookies_result, country_code)
                print(f"**🍪 Cookies generadas para {domain}**:\n{converted_cookies}\n🟢")
            else:
                print(f"**❌ No se pudieron generar cookies para {domain}**")
            await asyncio.sleep(5)  # Pausa entre dominios para evitar bloqueos

# Función principal
def main():
    print("🍪 Bot de Cookies Amazon - Versión Terminal 🍪")
    print("=" * 50)
    start()

    while True:
        print("\n" + "=" * 50)
        print("MENÚ PRINCIPAL:")
        print("1. Generar cookies para un país específico")
        print("2. Generar cookies para todos los países")
        print("3. Salir")
        print("=" * 50)

        try:
            opcion = input("Selecciona una opción (1-3): ").strip()

            if opcion == "1":
                print("\nPaíses disponibles:")
                print("CA (Canadá), MX (México), US (Estados Unidos), UK (Reino Unido)")
                print("DE (Alemania), FR (Francia), IT (Italia), ES (España)")
                print("JP (Japón), AU (Australia), IN (India)")

                country = input("Ingresa el código del país: ").strip().upper()
                if country:
                    asyncio.run(cookies(country))
                else:
                    print("**❌ Código de país vacío**")

            elif opcion == "2":
                confirmacion = input("¿Generar cookies para TODOS los países? Esto puede tomar tiempo (s/n): ").strip().lower()
                if confirmacion == "s" or confirmacion == "si":
                    asyncio.run(cookies())
                else:
                    print("Operación cancelada.")

            elif opcion == "3":
                print("👋 ¡Hasta luego!")
                break

            else:
                print("**❌ Opción no válida. Intenta de nuevo.**")

        except KeyboardInterrupt:
            print("\n👋 ¡Hasta luego!")
            break
        except Exception as e:
            print(f"**❌ Error: {str(e)}**")

if __name__ == '__main__':
    main()
