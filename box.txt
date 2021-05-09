package mvc;

import java.io.Serializable;

public class Box implements Serializable {
    //координаты бокса х и y на поле
    private int x;
    private int y;
    private Picture picture;  //значение соответствующей картинки для даннго бокса
    private boolean isOpen = false;  //флаг открыт ли данный бокс (== стрелял ли соперник в данный бокс)

    public boolean isOpen() {
        return isOpen;
    }

    public void setOpen(boolean open) {
        isOpen = open;
    }

    public Picture getPicture() {
        return picture;
    }

    public void setPicture(Picture picture) {
        this.picture = picture;
    }

    public Box(Picture picture, int x, int y) {
        this.picture = picture;
        this.x = x;
        this.y = y;
    }

    public int getX() {
        return x;
    }

    public int getY() {
        return y;
    }
}