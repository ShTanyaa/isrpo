# Крестики-нолики

Реализуем класс, имитирующий поле для игры в крестики-нолики и проверим его в консольном приложении.
Используем данный класс для создания приложения с графическим интерфейсом.

## 1. Основная идея

Для представления поля мы будем использовать двумерный массив целых чисел (3х3). В качестве крестика возьмем `1`, в качестве нолика `-1`, в качестве пустой ячейки `0`. Такой подход упростит проверку на окончание игры: мы можем посчитать сумму интересующей нас тройки значений (строка, столбец, диагональ) и проверить ее на равенство `3` или `-3` (соответственно победа крестика или нолика).

```
 0  0  0
 1  1  1  => сумма == 3, крестик победил
-1 -1  0
```

## 1.1 Использование enum

Для удобства, мы будем использовать вместо целых чисел значения перечисляемого типа `XOElement`:
```cs
public enum XOElement { None = 0, Cross = 1, Circle = -1 }
```

## 2. Инкапсуляция

Создадим консольное приложение. В отдельном файле создадим класс:
```cs
public class XOField
{

}
```

Добавим ему поле, представляющее двумерный массив:
```cs
	public class XOField
    {
        private XOElement[,] rawValues = new XOElement[3, 3];  
    }
```

Избавимся от магических чисел `3`, определив для них константы:
```cs
	public class XOField
    {
        private const int rowCount = 3, colCount = 3;
        private int[,] rawValues = new int[rowCount, colCount]
    }
```

> **Напоминание**
> `const` и `readonly` позволяют некоторому значению оставаться неизменным. 
> Отличие: значения с модификатором `readonly` устанавливаются во время *выполнения программы*, например, при вызове конструктора или инициализации поля. Значения с модификатором `const` определяются в *момент компиляции* и не могут быть изменены никаким способом (почти что).

Будет не очень хорошо, если пользователь нашего класса сможет напрямую задавать значения двумерного массива - он запросто сможет достичь недопустимого состояния нашего объекта (скажем, в поле будут одни нолики или что-нибудь еще такое).

Чтобы ограничить изменение исходного массива, **мы разрешим работать с ним только лишь через код описанных нами методов и свойств**. Подобное объединение данных и кода называют **инкапсуляцией**.

Во-первых, мы определили поле, как приватное (скрытое).

Во-вторых, для измения значения мы реализуем метод `TryTurn`. Логика следующая: если ход возможен, то мы учтем предыдущий ход и поставим нужный символ в указанную клетку. В случае успешного хода вернем `true`.

```cs
public class XOField
    {
        private const int rowCount = 3, colCount = 3;
        private XOElement[,] rawValues = new XOElement[rowCount, colCount];
        // последний ход, установим изначально в нолик, поскольку крестик ходит первым
        private XOElement lastTurn = XOElement.Circle; 
        
        private XOElement reverseElement(XOElement element)
        {
            // смена элемента на противоположный:
            // приводим в int, меняем знак, приводим обратно
            return (XOElement)(-(int)element);
        }

        public bool GameOver
        {
            get
            {
                // проверка конца игры
                throw new NotImplementedException();
            }
        }

        public bool CanTurn(int row, int col)
        {
            // можно ходить, если ячейка пуста и не наступил конец игры
            return rawValues[row, col] == XOElement.None && !GameOver;
        }

        public bool TryTurn(int row, int col)
        {
            if (!CanTurn(row, col))
            {
                return false;
            }

            rawValues[row, col] = reverseElement(lastTurn);
            lastTurn = rawValues[row, col];

            return true;
        }
    }
```

В-третьих, организуем возможность чтения элементов в виде привычных символов. Для этого создадим свойство `Field`:
```cs
		public char[,] Field
        {
            get
            {
                char[,] result = new char[rowCount, colCount];
                for (int i = 0; i < rowCount; i++)
                {
                    for (int j = 0; j < colCount; j++)
                    {
                        if (rawValues[i, j] == XOElement.Cross)
                            result[i, j] = 'X';
                        else if (rawValues[i, j] == XOElement.Circle)
                            result[i, j] = 'O';
                        else
                            result[i, j] = '-';
                    }
                }
                return result;
            }
        }
```

Проверим в консольном приложении (свойство `GameOver` нужно будет временно изменить, поймете после запуска):
```cs
using ConsoleApp123;

XOField XOGame = new XOField();
XOGame.TryTurn(0, 0);
XOGame.TryTurn(1, 0);
XOGame.TryTurn(1, 1);
XOGame.TryTurn(1, 1);
XOGame.TryTurn(2, 2);

for (int i = 0; i < XOGame.Field.GetLength(0); i++)
{
    for (int j = 0; j < XOGame.Field.GetLength(1); j++)
    {
        Console.Write(XOGame.Field[i, j] +  " ");
    }
    Console.WriteLine();
}
```
Результат:
```
X - -
O X -
- - O
```

Таким образом, мы реализовали возможность изменять значения поля, без возможности осуществления "незаконных" в мире крестиков-ноликов действий.

## 3. Проверка на победителя

Добавим еще одно приватное поле:
```cs
private XOElement winner = XOElement.None;
```

А также реализуем свойство:
```cs
		public XOElement Winner
        {
            get
            {
                return winner;
            }

            private set
            {
                winner = value;
            }
        }
```

Изменение победителя будем производить при осуществлении хода:
```cs
		public bool TryTurn(int row, int col)
        {
            if (!CanTurn(row, col))
            {
                return false;
            }

            rawValues[row, col] = reverseElement(lastTurn);
            lastTurn = rawValues[row, col];
            Winner = checkWinner(); // некоторая проверка, не победил ли кто сейчас
            return true;
        }
```

Код проверки (частичный):
```cs
		private XOElement checkRows()
        {
            for (int i = 0; i < rowCount; i++)
            {
                int sum = 0;
                for (int j = 0; j < colCount; j++)
                {
                    sum += (int)rawValues[i, j];
                }
                if (sum == 3)
                {
                    return XOElement.Cross;
                }
                if (sum == -3)
                {
                    return XOElement.Circle;
                }
            }
            return XOElement.None;
        }

        private XOElement checkWinner()
        {
            var winner = checkRows();
            if (winner != XOElement.None)
            {
                return winner;
            }
            // winner = checkCols();
            // ...
            // winner = checkDiagonals();
            // ...
            return XOElement.None;
        }
```

В примере выше реализована только лишь логика проверки строк. Реализуйте полноценную проверку на наличие победителя **самостоятельно**. Используйте тот факт, что если сумма равна 3, то побеждает крестик, если -3, то нолик. В принципе, можно придерживаться шаблона, написанного выше, то есть реализовать методы `checkCols` и `checkDiagonals`, но это не обязательно.

Также, не забудьте изменить свойство `GameOver`:
```cs
        public bool GameOver
        {
            get
            {
                return winner != XOElement.None;
            }
        }
```

Если все сделано правильно, то в случаях, вроде:
```cs
XOField XOGame = new XOField();
XOGame.TryTurn(0, 0);
XOGame.TryTurn(1, 0);
XOGame.TryTurn(0, 1);
XOGame.TryTurn(1, 1);
XOGame.TryTurn(0, 2);
XOGame.TryTurn(1, 2);

for (int i = 0; i < XOGame.Field.GetLength(0); i++)
{
    for (int j = 0; j < XOGame.Field.GetLength(1); j++)
    {
        Console.Write(XOGame.Field[i, j] +  " ");
    }
    Console.WriteLine();
}
```
Результат будет:
```
X X X
O O -
- - -
```

То есть последий ход `XOGame.TryTurn(1, 2)` не будет проделан, так как игра к тому времени закончится.

## 4. Добавление событий
Добавим два события. Одно будет вызвано при окончании игры. Другое - при каждом успешном ходе. Начнем со второго.

Сперва добавим в класс необходимый элемент:
```cs
public event Action<int, int, XOElement> OnTurn;
```

`Action<int, int, XOElement>` означение метод, который возвращает `void` и имеет три параметра: `int, int, XOElement`


Далее, модифицируем метод `TryTurn`:
```cs
        public bool TryTurn(int row, int col)
        {
            if (!CanTurn(row, col))
            {
                return false;
            }

            rawValues[row, col] = reverseElement(lastTurn);
            lastTurn = rawValues[row, col];

            if (OnTurn != null)
            {
                OnTurn.Invoke(row, col, lastTurn);
            }

            Winner = checkWinner();
            return true;
        }
```

Строки
```cs
			if (OnTurn != null)
            {
                OnTurn.Invoke(row, col, lastTurn);
            }
```
говорят о том, что если обработчик события определен (не равен null), то мы его вызовем.

C# позволяет описать проверку на `null` более лаконично:
```cs
        public bool TryTurn(int row, int col)
        {
            if (!CanTurn(row, col))
            {
                return false;
            }

            rawValues[row, col] = reverseElement(lastTurn);
            lastTurn = rawValues[row, col];
            // то же самое, но используем оператор ?
            OnTurn?.Invoke(row, col, lastTurn);
            Winner = checkWinner();
            return true;
        }
```

Попробуем описать обработчик в нашем консольном приложении:
```cs
XOField XOGame = new XOField();

XOGame.OnTurn += (row, col, elem) =>
{
    for (int i = 0; i < XOGame.Field.GetLength(0); i++)
    {
        for (int j = 0; j < XOGame.Field.GetLength(1); j++)
        {
            Console.Write(XOGame.Field[i, j] + " ");
        }
        Console.WriteLine();
    }
    Console.WriteLine("========");
};

XOGame.TryTurn(0, 0);
XOGame.TryTurn(1, 0);
XOGame.TryTurn(0, 1);
XOGame.TryTurn(1, 1);
XOGame.TryTurn(0, 2);
XOGame.TryTurn(1, 2);
```

Теперь вывод поля будет происходить каждый раз, когда происходит успешный ход. Заметьте, что после окончания игры ход не происходит, и вывод значений не произойдет.

Опишем еще одно событие - конец игры. Добавим необходимый элемент (`event`) в класс:
```cs
public event Action<XOElement> OnGameOver;
```

Вызовем обработчик внутри геттера свойства `Winner`:
```cs
        public XOElement Winner
        {
            get
            {
                return winner;
            }

            set
            {
                winner = value;
                // событие: кто-то победил
                if (winner != XOElement.None)
                {
                    OnGameOver?.Invoke(winner);
                }
            }
        }
```

Консольное приложение:
```cs
XOField XOGame = new XOField();

XOGame.OnTurn += (row, col, elem) =>
{
    for (int i = 0; i < XOGame.Field.GetLength(0); i++)
    {
        for (int j = 0; j < XOGame.Field.GetLength(1); j++)
        {
            Console.Write(XOGame.Field[i, j] + " ");
        }
        Console.WriteLine();
    }
    Console.WriteLine("========");
};

XOGame.OnGameOver += (winner) =>
{
    Console.WriteLine("GAME OVER!");
    if (winner == XOElement.Cross)
    {
        Console.WriteLine("X WIN");
    }
    else if (winner == XOElement.Circle)
    {
        Console.WriteLine("O WIN");
    }
};

XOGame.TryTurn(0, 0);
XOGame.TryTurn(1, 0);
XOGame.TryTurn(0, 1);
XOGame.TryTurn(1, 1);
XOGame.TryTurn(0, 2);
XOGame.TryTurn(1, 2);
```

В момент окончания игры будет вызван заданный обработчик, и мы увидем сообщение о победителе.

## 5. Использование класса

Создадим приложение на WPF. Добавим в него наш класс (по-хорошему, следовало сделать библиотеку, написать тесты, однако, для упрощения, мы просто скопируем код).

Зададим окну размеры, `Title` и запретим его масштабировать:
```xml
<Window x:Class="XODemo.MainWindow"
        xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation"
        xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
        xmlns:d="http://schemas.microsoft.com/expression/blend/2008"
        xmlns:mc="http://schemas.openxmlformats.org/markup-compatibility/2006"
        xmlns:local="clr-namespace:XODemo"
        mc:Ignorable="d"
        
        ResizeMode="NoResize"
        Title="XO Game" Height="600" Width="540">
```

Добавим в окно меню и canvas. Разместим их внутри элемента `DockPanel`, меню "прикреем" к верхней границе, canvas к нижней (используем `DockPanel.Dock`).
```xml
<DockPanel>
        <Menu DockPanel.Dock="Top">
            <MenuItem Header="File">
                <MenuItem Header="New game" />
                <Separator />
                <MenuItem Header="Exit" />
            </MenuItem>
            <MenuItem Header="?">
                <MenuItem Header="Об авторах" />
            </MenuItem>
        </Menu>
        
        <Canvas Background="Transparent"
            DockPanel.Dock="Bottom" x:Name="gameField" Margin="10" Width="500" Height="500"  >

        </Canvas>
    </DockPanel>
```

В коде окна (конструктор `MainWindow`) реализуем отрисовку прямоугольников (поле для игры):
```cs
        // размеры одной ячейки
        private readonly double rectWidth;
        private readonly double rectHeight;

        public MainWindow()
        {
            InitializeComponent();
            // берем 1/3 от размеров всего поля
            rectWidth = gameField.Width / 3;
            rectHeight = gameField.Height / 3;

            // координата y левого верхнего угла текущей ячейки
            double y = 0;

            // три строки
            for (int i = 1; i <= 3; i++)
            {
                // координата x левого верхнего угла текущей ячейки
                double x = 0;

                // три ячейки в каждой строке
                for (int j = 1; j <= 3; j++)
                {
                    // создаем прямоугольник
                    var rect = new Rectangle();
                    rect.Stroke = Brushes.Black;
                    rect.StrokeThickness = 1;
                    rect.Height = rectHeight;
                    rect.Width = rectWidth;
                    // ставим ему координаты
                    Canvas.SetLeft(rect, x);
                    Canvas.SetTop(rect, y);
                    // добавляем на canvas
                    gameField.Children.Add(rect);
                    // уходим правее для отрисовки следующей ячейки
                    x += rectWidth;
                }
                // уходим ниже для отрисовки следующей строки
                y += rectHeight;
            }
        }
```

Результат должен получиться примерно таким:
![](1.png)

Реализуем также два вспомогательных метода:
```cs
 		// считает по координатам строку и стобец ячейки
        private (int, int) calcCell(double x, double y)
        {

            int row = (int)(y / rectHeight);
            int col = (int)(x / rectWidth);

            return (row, col);
        }

        // вывод символа в указанную ячейку
        private void printSymbol(int row, int col, char symbol)
        {
           
            (double cornerX, double cornerY) = (rectHeight * col, rectWidth * row);

            Label text = new Label();
            text.Content = symbol;
            text.FontSize = rectWidth / 1.5;

            Canvas.SetLeft(text, cornerX + rectWidth / 4);
            Canvas.SetTop(text, cornerY);
            gameField.Children.Add(text);
        }
```

Добавим Canvas обработчик `MouseDown="makeTurn"`:
```cs
        private void makeTurn(object sender, MouseButtonEventArgs e)
        {
		    (int row, int col) = calcCell(e.GetPosition(gameField).X, e.GetPosition(gameField).Y);
            printSymbol(row, col, 'X');
        }
```

Теперь, при нажатии на ячейку, в ней будет отрисован символ `'X'`. Стоит запустить и проверить, что все идет по плану.

Если все хорошо, то самое время добавить в нашу программу экземпляр `XOField`. Приватное поле:
```cs
private readonly XOField XOGame = new XOField();
```

В конструктор `MainWindow` добавим строки:
```cs
			XOGame.OnTurn += (row, col, elem) =>
            {
                char symbol = elem == XOElement.Cross ? 'X' : 'O';
                printSymbol(row, col, symbol);
            };
```

Немного изменим обработчик `makeTurn`:
```cs
		private void makeTurn(object sender, MouseButtonEventArgs e)
        {
            (int row, int col) = calcCell(e.GetPosition(gameField).X, e.GetPosition(gameField).Y);
            XOGame.TryTurn(row, col);
        }
```

Теперь, игра практически готова.

**Самостоятельно**:
- реализуйте уведомление об окончании игры;
- реализуйте пункты меню "New Game" и "Exit". Возможно, вам понадобиться добавить в класс `XOField` какой-нибудь метод, вроде `Clear` или `Reset`;

**Дополнительные задания** (опциональное выполнение):
- реализуйте зачеркивание трех следующих подряд элементов при победе одного из игроков (как при игре на листочке). Полезным может оказаться элемент `Line`;
- программа содержит возможность выхода за границы массива. Попробуйте найти ошибку и исправить ее;
- для `Canvas`, используя событие `MouseMove` и свойство `Cursor`, а также используя свойство `XOField` `CanTurn`, добейтесь, чтобы курсор менялся в зависимости о того, возможен ли ход для определенной ячейки. Кстати, успешная реализация данной функции может помочь обнаружить ошибку из предыдущего пункта;
- модифицируйте программу так, чтобы вместо второго игрока играл ИИ. Для упрощения задачи, ход противника будет полностью основан на генераторе псевдослучайных чисел `Random`.