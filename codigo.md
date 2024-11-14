
```
using System.IO.Ports;
using Microsoft.ML;
using Microsoft.ML.Transforms.TimeSeries;





namespace Monitor
{

    public partial class Form1 : Form
    {
        private SerialPort serialPort;
        private System.Windows.Forms.Timer timer;
        private Graphics g;
        private Bitmap bmp;
        private int[] datos;
        private ITransformer bpmPredictionModel;
        private PredictionEngine<BpmDatos, BpmPrediccion> predictionEngine;

        private bool panelExpandido = false;
        private const int panelAlturaMax = 200;
        private const int panelAlturaMin = 0;

        public Form1()
        {
            InitializeComponent();
        }

        private void Form1_Load(object sender, EventArgs e)
        {

            TrainModel();
            if (bpmPredictionModel != null)
            {
                predictionEngine = new MLContext().Model.CreatePredictionEngine<BpmDatos, BpmPrediccion>(bpmPredictionModel);
            }
            else
            {
                MessageBox.Show("El modelo no se entrenó correctamente.");
            }

            serialPort = new SerialPort("COM3", 9600);
            try
            {
                serialPort.Open();
                serialPort.DataReceived += SerialPort_DataReceived;
            }
            catch (UnauthorizedAccessException)
            {
                MessageBox.Show("El puerto está en uso por otro programa. Cierre cualquier otro programa que esté utilizando el puerto COM5.");
            }
            catch (IOException ex)
            {
                MessageBox.Show($"Error de conexión: {ex.Message}");
            }
            catch (Exception ex)
            {
                MessageBox.Show($"Error: {ex.Message}");
            }

            timer = new System.Windows.Forms.Timer();
            timer.Interval = 50;
            timer.Tick += Timer_Tick;
            timer.Start();


            bmp = new Bitmap(panelMonitor.Width, panelMonitor.Height);
            g = Graphics.FromImage(bmp);


            datos = new int[panelMonitor.Width];
        }
        private void Timer_Tick(object sender, EventArgs e)
        {
            g.Clear(Color.Black);


            for (int i = 1; i < datos.Length; i++)
            {
                int x1 = i - 1;
                int y1 = panelMonitor.Height - datos[i - 1];
                int x2 = i;
                int y2 = panelMonitor.Height - datos[i];

                g.DrawLine(Pens.Green, x1, y1, x2, y2);
            }


            panelMonitor.Image = bmp;
        }
        private List<float> bpmHistory = new List<float>(); // Lista para mantener el historial de BPM

        private void SerialPort_DataReceived(object sender, SerialDataReceivedEventArgs e)
        {
            string data = serialPort.ReadLine();
            if (int.TryParse(data, out int tiempoTranscurrido))
            {
                int bpm = tiempoTranscurrido > 0 ? 60000 / tiempoTranscurrido : 0;

                this.Invoke((MethodInvoker)delegate
                {
                    labelBPM.Text = $"BPM: {bpm}";
                    bpmHistory.Add(bpm);

                    if (bpmHistory.Count > 100) // Limitar el tamaño del historial
                    {
                        bpmHistory.RemoveAt(0);
                    }
                });

                // Asegúrate de que el historial tiene suficientes datos para predecir
                if (bpmHistory.Count >= 10)
                {
                    // Crear un objeto con el historial para pasarlo al modelo
                    var bpmData = new BpmDatos { BPM = bpm };
                    AnalizarBPM(bpmData);
                }
            }
        }
        private void label3_Click(object sender, EventArgs e) { }
        private void panel1_Click(object sender, EventArgs e) { }
        private void panelMonitor_Click(object sender, EventArgs e) { }
        private void button1_Click(object sender, EventArgs e) { }
        private void capturaP_Click(object sender, EventArgs e)
        {

        }
        private void AnalizarBPM(BpmDatos bpmData)
        {
            // Verificar que predictionEngine no sea nulo
            if (predictionEngine == null)
            {
                MessageBox.Show("Prediction engine no está inicializado.");
                return;
            }

            var prediction = predictionEngine.Predict(bpmData);

            if (prediction.Anormalidad)
            {
                MessageBox.Show("Anomalía detectada en el ritmo cardíaco.");
            }
        }


            private void TrainModel()
        {
            var mlContext = new MLContext();

            // Ejemplo de datos más representativos
            var trainingData = new List<BpmDatos>
    {
        new BpmDatos { BPM = 60 },
        new BpmDatos { BPM = 70 },
        new BpmDatos { BPM = 75 },
        new BpmDatos { BPM = 80 },
        new BpmDatos { BPM = 85 },
        new BpmDatos { BPM = 90 },
        new BpmDatos { BPM = 95 },
        new BpmDatos { BPM = 100 },
        // Añadir más datos para hacer el modelo más robusto
    };

            var trainingDataView = mlContext.Data.LoadFromEnumerable(trainingData);

            var pipeline = mlContext.Transforms.DetectAnomalyBySrCnn(
                outputColumnName: nameof(BpmPrediccion.Anormalidad),
                inputColumnName: nameof(BpmDatos.BPM),
                windowSize: 100,
                backAddWindowSize: 5,
                threshold: 0.3);

            bpmPredictionModel = pipeline.Fit(trainingDataView);
        }



    }


    public class BpmDatos
    {
        public float BPM { get; set; }
    }

    public class BpmPrediccion
    {
        public bool Anormalidad { get; set; }
    }
}
