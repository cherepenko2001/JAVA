package mvc;

import mvc.connect.Connection;
import mvc.connect.Message;
import mvc.connect.MessageType;
import mvc.connect.Server;

import java.io.IOException;
import java.net.Socket;
import java.util.List;

public class Controller {
    private View view;
    private Model model;
    private Connection connection;

    public Controller(View view, Model model) {
        this.view = view;
        this.model = model;
    }

    //загружает наше пустое игровое поле
    public void loadEmptyMyField() {
        //очистка списков всех типов кораблей
        model.getShipsOneDeck().clear();
        model.getShipsTwoDeck().clear();
        model.getShipsThreeDeck().clear();
        model.getShipsFourDeck().clear();
        model.setMyField(new Box[Picture.COLUMNS][Picture.ROWS]);
        //присваение элементам матрицы нашего игрового поля значений
        for (int i = 0; i < Picture.ROWS; i++) {
            for (int j = 0; j < Picture.COLUMNS; j++) {
                if (i == 0 && j == 0) continue;
                else if (i == 0 && j != 0) { //если это первый столбец то присваиваем значение картинок с буквами
                    model.addBoxInField(model.getMyField(), new Box(Picture.valueOf("SYM" + j), Picture.IMAGE_SIZE * i, Picture.IMAGE_SIZE * j));
                } else if (i != 0 && j == 0) { //если это первая строка присваиваем значение картинок с цифрами
                    model.addBoxInField(model.getMyField(), new Box(Picture.valueOf("NUM" + i), Picture.IMAGE_SIZE * i, Picture.IMAGE_SIZE * j));
                } else { //в остальных случаях знаяение картинки с пустой клеткой
                    model.addBoxInField(model.getMyField(), new Box(Picture.EMPTY, Picture.IMAGE_SIZE * i, Picture.IMAGE_SIZE * j));
                }
            }
        }
    }

    //добавляет корабль предварительно проверяя - не пересекается ли новый корабль с уже добавленным ораблем
    public void addShip(Ship ship) {
        List<Box> boxesOfShip = ship.getBoxesOfShip();
        for (Box boxShip : boxesOfShip) {
            if (checkAround(boxShip, boxesOfShip)) {
                boxesOfShip.clear();
                return;
            }
        }
        if (boxesOfShip.size() != 0) model.addShip(ship);
    }

    //удаляет корабль по заданным координатам
    public Ship removeShip(int x, int y) {
        List<Ship> allShips = model.getAllShips(); //получаем список всех добавленных кораблей
        for (Ship ship : allShips) {
            for (Box box : ship.getBoxesOfShip()) {
                if (x == box.getX() && y == box.getY()) { //перебираем корабли, затем их боксы,
                    // если координаты бокса совпадают с заданными - удаляем корабль
                    model.removeShip(ship);
                    return ship;
                }
            }
        }
        return null;
    }

    //метод проверки на пересечение с другими короблями
    private boolean checkAround(Box box, List<Box> boxesOfShip) {
        int myX = box.getX();
        int myY = box.getY();
        for (int i = myX - Picture.IMAGE_SIZE; i <= myX + Picture.IMAGE_SIZE; i += Picture.IMAGE_SIZE) {
            for (int j = myY - Picture.IMAGE_SIZE; j <= myY + Picture.IMAGE_SIZE; j += Picture.IMAGE_SIZE) {
                Box boxFromMatrix = model.getBox(model.getMyField(), i, j);
                if (boxFromMatrix != null && boxFromMatrix.getPicture() == Picture.SHIP && !boxesOfShip.contains(boxFromMatrix)) {
                    View.callInformationWindow("Сюда нельзя добавлять корабль - пересечение с другим");
                    boxesOfShip.clear();
                    return true;
                }
            }
        }
        return false;
    }

    //метод открывающий путсые клетки вокруг подбитого корабля на поле противника
    public void openBoxesAround(Box boxShot) {
        Ship ship = model.getShipOfEnemy(boxShot); //по боксу в который выстрелили получаем корабль
        if (ship != null) {
            //если число палуб == числу подбитых (открытых) боксов корабля - то открываем все пустые клетки вокруг
            if (ship.getCountDeck() == getCountOpenBoxOfShip(ship)) model.openAllBoxesAroundShip(ship);
                //иначе если подбита (отрыт бокс) только одна палуба - ничего не делаем
            else if (getCountOpenBoxOfShip(ship) == 1) return;
                // в остальных случаях открываем пустые клетки вокруг подбитых палуб
            else {
                for (Box box : ship.getBoxesOfShip()) {
                    if (box.isOpen())
                        model.openBoxAroundBoxOfShipEnemy(box.getX(), box.getY(), ship.isHorizontalPlacement());
                }
            }
        }
    }

    //метод возвращает количество открытых боксов (подбитых палуб) корабля
    public int getCountOpenBoxOfShip(Ship ship) {
        int count = 0;
        for (Box box : ship.getBoxesOfShip()) {
            if (box.isOpen()) count++;
        }
        return count;
    }

    //проверка на окончание игры
    private boolean checkEndGame() {
        List<Box> allBoxesOfShip = model.getAllBoxesOfShips(); //получаем список всех своих кораблей
        for (Box box : allBoxesOfShip) {
            //проверяем сли хотя бы один корабль имеет значение картини SHIP то игра не окончена
            if (box.getPicture() == Picture.SHIP) return false;
        }
        return true;
    }

    //проверка на полный комплект добавленный кораблей перед стартом игры
    public boolean checkFullSetShips() {
        //просто проверяем количство корабля в соответствующих списках по каждому типу кораблей
        if (model.getShipsOneDeck().size() == 4 &&
                model.getShipsTwoDeck().size() == 3 &&
                model.getShipsThreeDeck().size() == 2 &&
                model.getShipsFourDeck().size() == 1) return true;
        else return false;
    }

    //ДАЛЕЕ ИДУТ МЕТОДЫ ДЛЯ КОННЕКТА К СЕРВЕРУ И ОБМЕНА ВЫСТРЕЛАМИ МЕЖДУ ИГРОКАМИ

    //создает игровую комнату
    public void createGameRoom(int port) throws IOException {
        Server server = new Server(port); //создаем объект класса сервер и запускаем поток исполнения
        server.start();
    }


    //метод подключения клиента к серверу
    public void connectToRoom(int port) throws IOException, ClassNotFoundException {
        connection = new Connection(new Socket("localhost", port));
        Message message = connection.receive(); //принимае от сервера сообщение
        //если тип сообщения ACCEPTED то отправляем на сервер наше поле с кораблями и список всех кораблей
        if (message.getMessageType() == MessageType.ACCEPTED) {
            connection.send(new Message(MessageType.FIELD, model.getMyField(), model.getAllShips()));
            Message messageField = connection.receive(); //ждем ответа от сервера с полем и списком кораблей соперника
            if (messageField.getMessageType() == MessageType.FIELD) {
                model.setEnemyField(messageField.getGameField());  //сохраняем в в модель поле и список кораблей противника
                model.setAllShipsOfEnemy(messageField.getListOfAllShips());
            }
        }
    }

    //метод отключения от сервера (выход из игрвоой комнаты)
    public void disconnectGameRoom() throws IOException {
        connection.send(new Message(MessageType.DISCONNECT));
    }

    //отправка сообщения на сервер с координатами выстрела
    public boolean sendMessage(int x, int y) throws IOException {
        Box box = model.getBox(model.getEnemyField(), x, y);
        if (!box.isOpen()) {
            box.setOpen(true); //открываем бокс выстрела
            openBoxesAround(box); //открываем соседние пустые клетки (боксы)
            connection.send(new Message(MessageType.SHOT, x, y)); //отправляем координаты выстрела на сервер
            return true;
        } else return false;
    }

    //метод принимающий сообщения от сервера
    public boolean receiveMessage() throws IOException, ClassNotFoundException {
        Message message = connection.receive(); //принимаем сообщение
        if (message.getMessageType() == MessageType.SHOT) { //если это выстрел, то...
            int x = message.getX();
            int y = message.getY();
            Box box = model.getBox(model.getMyField(), x, y); //получаем бокс с нашего поля по координатам
            //если бокс с картинкой "пустая клетка", то устанавливаем значение "пустая клетка с точкой"
            if (box.getPicture() == Picture.EMPTY) {
                box.setPicture(Picture.POINT);
                // если бокс с картинкой "корабль" то устанавливаем значение "уничтоженный корабль"
            } else if (box.getPicture() == Picture.SHIP) {
                box.setPicture(Picture.DESTROY_SHIP);
            }
            model.addBoxInField(model.getMyField(), box); //устанавливаем измененный бокс в матрицу нашего поля
            if (checkEndGame()) { //проверка на конец игры, если да послыем соответствующее сообщение
                connection.send(new Message(MessageType.DEFEAT));
                View.callInformationWindow("Вы проиграли! Все Ваши корабли уничтожены.");
                return false;
            }
            return true;
            //если тип сообщения DISCONNECT отправялем, что мы тоже отключаемся
        } else if (message.getMessageType() == MessageType.DISCONNECT) {
            connection.send(new Message(MessageType.MY_DISCONNECT));
            View.callInformationWindow("Ваш соперник покинул игру. Вы одержали техническую победу!");
            return false;
            //если тип сообщения DEFEAT отправялем, что мы тоже отключаемся
        } else if (message.getMessageType() == MessageType.DEFEAT) {
            connection.send(new Message(MessageType.MY_DISCONNECT));
            View.callInformationWindow("Все корабли противника уничтожены. Вы одержали победу!");
            return false;
        } else return false;
    }
}