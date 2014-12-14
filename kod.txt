using System;
using System.Text;
using System.Net;
using System.Net.Sockets;
using System.Threading;
using Microsoft.SPOT;
using Microsoft.SPOT.Hardware;
using SecretLabs.NETMF.Hardware;
using SecretLabs.NETMF.Hardware.NetduinoPlus;
using Socket = System.Net.Sockets.Socket;
using System.IO;


namespace NetduinoApplication1
{


    public class Program
    {
        static OutputPort led = new OutputPort(Pins.ONBOARD_LED, false);

        static Receiver receiver = new Receiver("184.106.153.149", 80);



        //ThingSpeak Settings
        const string writeAPIKey = "Kendi API kodunuzu yazın"; // Write API Key for a ThingSpeak Channel
        static string tsIP = "184.106.153.149";     // IP Address for the ThingSpeak API
        static Int32 tsPort = 80;                   // Port Number for ThingSpeak
        const int updateInterval = 10000;           // Time interval in milliseconds to update ThingSpeak 


        static PWM servo = new PWM(Pins.GPIO_PIN_D5);

        // Servonun gidebileceği açıları sınırlandır (0 - 180 derece)
        static short servoMinPosition = 10, servoMaxPosition = 160;
        public static void Main()
        {
            OneWire oneWire = new OneWire(new OutputPort(Pins.GPIO_PIN_D0, false));


            Debug.Print("App started!");
            ushort temperature;
            while (true)
            {
                
                if (true)
                {
                    
                    oneWire.WriteByte(0xCC); // Skip ROM, we only have one device
                    oneWire.WriteByte(0x44); // Start temperature conversion
                    while (oneWire.ReadByte() == 0) ; // wait while busy
                    oneWire.TouchReset();
                    oneWire.WriteByte(0xCC); // Skip ROM
                    oneWire.WriteByte(0xBE); // Read Scratchpad
                    temperature = (byte)oneWire.ReadByte(); // LSB
                    temperature |= (ushort)(oneWire.ReadByte() << 8); // MSB
                    

                    
                   

                    Debug.Print("Receiver:");
                    string received = receiver.Reveive("/channels/"kendi kanal adresiniz"/fields/field1/last");
                    // Debug.Print(received);
                 string result = received.Substring(4, 6);
                    if (result == "doldur")
                    {
                        Debug.Print("kova dolduruldu");
                    }
                    if (result == "bosalt")
                    {

                        // Servo 111 derece
                        SetServoPosition(111);
                        led.Write(true);
                        Thread.Sleep(2000);

                        // Servo 180 derece
                        SetServoPosition(180);
                        led.Write(false);
                       
                        
                        Debug.Print("kova boşaltıldı");

                        delayLoop(updateInterval);

                        updateThingSpeak("field1= doldur");


                    }
        
                }

            }
        }
        private static void SetServoPosition(short position)
        {
            //birsaniye neyi sildin :) yanış kank 
            if (position < servoMinPosition) position = servoMinPosition;
            else if (position > servoMaxPosition) position = servoMaxPosition;
            servo.SetPulse(20000, 500 + (uint)(position * 11.11));
        }
        static void updateThingSpeak(string tsData)
        {
            //            Debug.Print("Connected to ThingSpeak...\n");


            String request = "POST /update HTTP/1.1\n";
            request += "Host: api.thingspeak.com\n";
            request += "Connection: close\n";
            request += "X-THINGSPEAKAPIKEY: " + writeAPIKey + "\n";
            request += "Content-Type: application/x-www-form-urlencoded\n";
            request += "Content-Length: " + tsData.Length + "\n\n";

            request += tsData;

            try
            {
                String tsReply = sendPOST(tsIP, tsPort, request);
                //    Debug.Print(tsReply);
                //   Debug.Print("...disconnected.\n");

            }
            catch (SocketException se)
            {
                Debug.Print("Connection Failed.\n");
                Debug.Print("Socket Error Code: " + se.ErrorCode.ToString());
                Debug.Print(se.ToString());
                Debug.Print("\n");

            }
        }

        // Issues a http POST request to the specified server. (From the .NET Micro Framework SDK example)
        private static String sendPOST(String server, Int32 port, String request)
        {
            const Int32 c_microsecondsPerSecond = 1000000;

            // Create a socket connection to the specified server and port.
            using (Socket serverSocket = ConnectSocket(server, port))
            {
                // Send request to the server.
                Byte[] bytesToSend = Encoding.UTF8.GetBytes(request);
                serverSocket.Send(bytesToSend, bytesToSend.Length, 0);

                // Reusable buffer for receiving chunks of the document.
                Byte[] buffer = new Byte[1024];

                // Accumulates the received page as it is built from the buffer.
                String page = String.Empty;

                // Wait up to 30 seconds for initial data to be available.  Throws an exception if the connection is closed with no data sent.
                DateTime timeoutAt = DateTime.Now.AddSeconds(30);
                while (serverSocket.Available == 0 && DateTime.Now < timeoutAt)
                {
                    System.Threading.Thread.Sleep(100);
                }

                // Poll for data until 30-second timeout.  Returns true for data and connection closed.
                while (serverSocket.Poll(30 * c_microsecondsPerSecond, SelectMode.SelectRead))
                {
                    // If there are 0 bytes in the buffer, then the connection is closed, or we have timed out.
                    if (serverSocket.Available == 0) break;

                    // Zero all bytes in the re-usable buffer.
                    Array.Clear(buffer, 0, buffer.Length);

                    // Read a buffer-sized HTML chunk.
                    Int32 bytesRead = serverSocket.Receive(buffer);

                    // Append the chunk to the string.
                    page = page + new String(Encoding.UTF8.GetChars(buffer));
                }

                // Return the complete string.
                return page;
            }
        }

        // Creates a socket and uses the socket to connect to the server's IP address and port. (From the .NET Micro Framework SDK example)
        private static Socket ConnectSocket(String server, Int32 port)
        {
            // Get server's IP address.
            IPHostEntry hostEntry = Dns.GetHostEntry(server);

            // Create socket and connect to the server's IP address and port
            Socket socket = new Socket(AddressFamily.InterNetwork, SocketType.Stream, ProtocolType.Tcp);
            socket.Connect(new IPEndPoint(hostEntry.AddressList[0], port));
            return socket;
        }

        static void delayLoop(int interval)
        {
            long now = DateTime.Now.Ticks / TimeSpan.TicksPerMillisecond;
            int offset = (int)(now % interval);
            int delay = interval - offset;
            Thread.Sleep(delay);
        }




    }
}

