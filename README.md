# Nixie_weather_report
uses open weather map to display fahrenheit temperatures to nixie tubes

import RPi.GPIO as GPIO
import time
import datetime
from datetime import datetime
from requests import get
from datetime import datetime
from json import loads
from pprint import pprint

PIN_DATA = 23
PIN_LATCH = 24
PIN_CLK = 25

GPIO.setwarnings(False)
GPIO.setmode(GPIO.BCM)
GPIO.setup(4, GPIO.IN)
GPIO.setup(17, GPIO.IN)
GPIO.setup(22, GPIO.IN)
GPIO.setup(18, GPIO.IN)
GPIO.setup(10, GPIO.IN)
GPIO.setup(9, GPIO.IN)
GPIO.setup(11, GPIO.IN)
GPIO.setup(27, GPIO.IN)
GPIO.setup(7, GPIO.OUT)
GPIO.setup(8, GPIO.OUT)


KEY = '0c04e5ddacbc68d164d7af4e9a487982'

class Nixie:
    def __init__(self, pin_data, pin_latch, pin_clk, digits):
        self.pin_data = pin_data
        self.pin_latch = pin_latch
        self.pin_clk = pin_clk
        self.digits = digits

        #GPIO.setmode(GPIO.BCM)

        # Setup the GPIO pins as outputs
        GPIO.setup(self.pin_data, GPIO.OUT)
        GPIO.setup(self.pin_latch, GPIO.OUT)
        GPIO.setup(self.pin_clk, GPIO.OUT)

        # Set the initial state of our GPIO pins to 0
        GPIO.output(self.pin_data, False)
        GPIO.output(self.pin_latch, False)
        GPIO.output(self.pin_clk, False)

    def delay(self):
        # We'll use a 10ms delay for our clock
        time.sleep(0.010)

    def transfer_latch(self):
        # Trigger the latch pin from 0->1. This causes the value that we've
        # been shifting into the register to be copied to the output.
        GPIO.output(self.pin_latch, True)
        self.delay()
        GPIO.output(self.pin_latch, False)
        self.delay()

    def tick_clock(self):
        # Tick the clock pin. This will cause the register to shift its
        # internal value left one position and the copy the state of the DATA
        # pin into the lowest bit.
        GPIO.output(self.pin_clk, True)
        self.delay()
        GPIO.output(self.pin_clk, False)
        self.delay()

    def shift_bit(self, value):
        # Shift one bit into the register.
        GPIO.output(self.pin_data, value)
        self.tick_clock()

    def shift_digit(self, value):
        # Shift a 4-bit BCD-encoded value into the register, MSB-first.
        self.shift_bit(value&0x08)
        value = value << 1
        self.shift_bit(value&0x08)
        value = value << 1
        self.shift_bit(value&0x08)
        value = value << 1
        self.shift_bit(value&0x08)

    def set_value(self, value):
        # Shift a decimal value into the register

        str = "%0*d" % (self.digits, value)

        for digit in str:
            self.shift_digit(int(digit))
            value = value * 10

        self.transfer_latch()

def get_city_id():
        with open('city.list.json') as f:
            data = loads(f.read())
        city = input('Which is the closest city to the place you are travelling to?' )
        city_id = False
        for item in data:
            if item['name'] == city:
                answer = input('Is this in ' + item['country'])
                if answer == 'y':
                        city_id = item['id']
                        break

        if not city_id:
                print('Sorry, that location is not available')
                exit()
                
        return city_id

def get_weather_data(city_id):
        weather_data = get('http://api.openweathermap.org/data/2.5/forecast?id={}&APPID={}'.format(city_id, KEY))
        return weather_data.json()

def get_arrival():
    today = datetime.now()
    #max_day = today + timedelta(days = 4)
    #print('What day of the month do you plan to arrive at your destination?')
    #print(today.strftime('%d'), '-', max_day.strftime('%d'))
    day = today.strftime('%d')
    hour = int(today.strftime('%H'))
    #print('The hour is: ',hour)
    #print('What hour do you plan to arrive?')
    #print('0 - 24')
    t = datetime.now()
    #hour = int(0)
    hour = hour - hour % 3
    hour = hour + 9
    if hour>24:
        newhour = hour - 24
        hour = newhour
        day = 1 + int(day)
    arrival = today.strftime('%Y') + '-' + today.strftime('%m') + '-' + str(day) + ' ' + str(hour) + ':00:00'
    #arrival = t
    return arrival

def get_forecast(arrival, weather_data):
    for forecast in weather_data['list']:
        if forecast['dt_txt'] == arrival:
            return forecast

def get_readable_forecast(forecast):
    weather = float(forecast['main']['temp']) #this now gives a single number for weather in kelvin
    return weather

def main(city):
    try:
        #city_id = get_city_id()
        #print(datetime.now())
        #asks user for city id
        #weather_data = get_weather_data(city_id)
        weather_data = get_weather_data(city)
        #pprint(weather_data)
        #uses city_id to get data
        arrival = get_arrival()
        #asks user for arrival time
        forecast = get_forecast(arrival, weather_data)
        #writes forecast
        weather = get_readable_forecast(forecast)
        #retrieves just the temp

        fahrenheit = weather - 273
        new_weather = (fahrenheit * 1.8) + 32
        newer_weather = round(new_weather, 1)
        #converts kelvin to fahrenheit
        print('the weather in fahrenheit is: ')
        pprint(newer_weather) 
        #pprint(time.asctime(time.localtime(time.time())))
        #prints weather in fahrenheit

        if newer_weather < 0:
            GPIO.setup(7, GPIO.HIGH)
            GPIO.setup(8, GPIO.HIGH)
        
        nixie = Nixie(PIN_DATA, PIN_LATCH, PIN_CLK, 3)
        nixie.set_value(newer_weather)
        time.sleep(5.00)
        nixie.set_value(000)
        GPIO.setup(7, GPIO.LOW)

    finally:
        #GPIO.cleanup()
        return

if __name__ == "__main__":
    #main()
    while True:
        if GPIO.input(4): #Aspen uses pin input 10
            print('Aspen')
            main(5412230)
            #break
        elif GPIO.input(17): #Copper uses pin input 12
            print('Copper')
            main(5422503)
            #break
        elif GPIO.input(11): #Eldora uses pin input 16
            print('Eldora')
            main(5432410)
            #break
        elif GPIO.input(22): #Steamboat uses pin input 18
            print('Steamboat')
            main(5582371)
            #break
        elif GPIO.input(18): #Vail uses pin input 22
            print('Vail')
            main(5442727)
            #break
        elif GPIO.input(10): #Telluride  uses pin input 24
            print('Telluride')
            main(5441199)
            #break
        elif GPIO.input(9): #Breckenridge uses pin input 26
            print('Breckenridge')
            main(4676181)
            #break
        elif GPIO.input(27): #Winter park uses pin input 3
            print('Winter Park')
            main(5425043)
            #break
        elif GPIO.input(11) & GPIO.input(17):
            GPIO.cleanup()
            
        
            

            

            
