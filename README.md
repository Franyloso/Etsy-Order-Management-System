-----***** main.cpp *****-----
#include "mainwindow.h"
#include <QApplication>

int main(int argc, char *argv[])
{
    QApplication app(argc, argv);
    MainWindow window;
    window.show();
    return app.exec();
}
-----***** mainwindow.cpp *****-----
#include "mainwindow.h"
#include "ui_mainwindow.h"
#include <QChart>
#include <QChartView>
#include <QBarSeries>
#include <QBarSet>
#include <QCategoryAxis>
#include <QValueAxis>
#include <QPushButton>
#include <QTextEdit>
#include <QTableWidget>
#include <QVBoxLayout>
#include <QScrollArea>
#include <QRegularExpression>
#include <QMap>
#include <QDebug>
#include <algorithm>  // For std::max_element

MainWindow::MainWindow(QWidget *parent)
    : QMainWindow(parent)
    , ui(new Ui::MainWindow)
{
    ui->setupUi(this);

    chartScrollArea = new QScrollArea(this);
    chartScrollArea->setWidgetResizable(true);

    // Initialize the QChartView
    chartView = new QChartView(this);
    chartView->setMinimumSize(1200, 800);  // Large size for better readability

    // Add the chartView to the scroll area
    chartScrollArea->setWidget(chartView);

    // Replace the layout of graphContainer with the scroll area
    QVBoxLayout *layout = new QVBoxLayout(ui->graphContainer);
    layout->addWidget(chartScrollArea);

    connect(ui->parseButton, &QPushButton::clicked, this, &MainWindow::loadOrderData);

    setupChart();
}

MainWindow::~MainWindow()
{
    delete ui;
}

void MainWindow::setupChart()
{
    // Create an empty chart and set its title
    QChart *chart = new QChart();
    chart->setTitle("Total Quantity Ordered Per Item");

    chartView->setChart(chart);
    chartView->setRenderHint(QPainter::Antialiasing);
}

void MainWindow::loadOrderData()
{
    // Retrieve raw order data from orderTextInput widget
    QString orderData = ui->orderTextInput->toPlainText();

    // Parse the order data
    QList<OrderInfo> orders = parseOrderText(orderData);

    // Debugging: Output parsed orders
    qDebug() << "Parsed Orders:" << orders.size();
    for (const auto &order : orders) {
        qDebug() << "Order ID:" << order.orderId << "Date:" << order.orderDate
                 << "Items:" << order.itemNames << "Quantities:" << order.quantities;
    }

    // Clear and configure the table for new data
    ui->orderTable->clear();
    ui->orderTable->setRowCount(0);  // Clear all rows
    ui->orderTable->setColumnCount(5);  // Columns: Order ID, Date, Item Name, Quantity, Profit
    ui->orderTable->setHorizontalHeaderLabels({"Order ID", "Date Ordered", "Item Name", "Quantity", "Profit ($)"});

    // Variables to store totals
    int totalQuantity = 0;
    double totalProfit = 0.0;

    // Populate the table with detailed data
    int row = 0;

    for (const OrderInfo &order : orders) {
        for (int i = 0; i < order.itemNames.size(); ++i) {
            QString itemName = order.itemNames[i];
            int quantity = order.quantities[i];

            // Determine the price per item
            double pricePerItem = 0.0;
            if (itemName.contains("Charm", Qt::CaseInsensitive)) {
                pricePerItem = 4.99;
            } else if (itemName.contains("Straw Cover", Qt::CaseInsensitive)) {
                pricePerItem = 5.99;
            } else if (itemName.contains("Keychain", Qt::CaseInsensitive)) {
                pricePerItem = 2.99;
            }

            // Calculate the order total and profit
            double orderTotal = pricePerItem * quantity;
            double profit = orderTotal - (orderTotal * 0.095);  // Subtract 9.5% fee

            // Update totals
            totalQuantity += quantity;
            totalProfit += profit;

            // Debugging: Verify table population data
            qDebug() << "Adding row:" << row
                     << "Order ID:" << order.orderId
                     << "Date:" << order.orderDate
                     << "Item:" << itemName
                     << "Quantity:" << quantity
                     << "Profit:" << profit;

            // Populate table
            ui->orderTable->insertRow(row);
            ui->orderTable->setItem(row, 0, new QTableWidgetItem(order.orderId));
            ui->orderTable->setItem(row, 1, new QTableWidgetItem(order.orderDate));  // Date Ordered column
            ui->orderTable->setItem(row, 2, new QTableWidgetItem(itemName));
            ui->orderTable->setItem(row, 3, new QTableWidgetItem(QString::number(quantity)));
            ui->orderTable->setItem(row, 4, new QTableWidgetItem(QString::number(profit, 'f', 2)));  // 2 decimal places

            row++;
        }
    }

    // Add totals row
    ui->orderTable->insertRow(row);
    ui->orderTable->setItem(row, 0, new QTableWidgetItem("Total"));
    ui->orderTable->setItem(row, 1, new QTableWidgetItem(""));  // Blank for Date Ordered
    ui->orderTable->setItem(row, 2, new QTableWidgetItem(""));  // Blank for Item Name
    ui->orderTable->setItem(row, 3, new QTableWidgetItem(QString::number(totalQuantity)));  // Total Quantity
    ui->orderTable->setItem(row, 4, new QTableWidgetItem(QString::number(totalProfit, 'f', 2)));  // Total Profit

    // Adjust column widths to fit content
    ui->orderTable->resizeColumnsToContents();

    // Bar chart logic (unchanged from previous code)
    QMap<QString, int> dateTotals;
    for (const OrderInfo &order : orders) {
        for (int i = 0; i < order.itemNames.size(); ++i) {
            int quantity = order.quantities[i];
            dateTotals[order.orderDate] += quantity;
        }
    }

    QBarSet *barSet = new QBarSet("Total Quantity");
    QStringList dateLabels;
    for (auto it = dateTotals.begin(); it != dateTotals.end(); ++it) {
        dateLabels.append(it.key());
        barSet->append(static_cast<int>(it.value()));  // Ensure integer values
    }

    QBarSeries *series = new QBarSeries();
    series->append(barSet);

    QChart *chart = chartView->chart();
    chart->removeAllSeries();
    chart->addSeries(series);

    QCategoryAxis *axisX = new QCategoryAxis();
    for (int i = 0; i < dateLabels.size(); ++i) {
        axisX->append(dateLabels.at(i), i);
    }
    axisX->setLabelsAngle(-45);
    axisX->setLabelsFont(QFont("Arial", 8));
    chart->setAxisX(axisX, series);

    QValueAxis *axisY = new QValueAxis();
    axisY->setTitleText("Total Quantity Ordered");
    axisY->setLabelFormat("%i");
    axisY->setRange(0, *std::max_element(dateTotals.begin(), dateTotals.end()));
    axisY->setTickCount(*std::max_element(dateTotals.begin(), dateTotals.end()) + 1);
    chart->setAxisY(axisY, series);

    chart->setMargins(QMargins(10, 10, 10, 10));
    chart->legend()->setVisible(true);
    chart->legend()->setAlignment(Qt::AlignBottom);
}

QList<MainWindow::OrderInfo> MainWindow::parseOrderText(const QString &text)
{
    QList<OrderInfo> orders;

    // Split orders by "Select this order"
    QStringList orderEntries = text.split(QRegularExpression(R"(\bSelect this order\b)"), Qt::SkipEmptyParts);

    for (const QString &entry : orderEntries) {
        OrderInfo order;

        // Regular expression for order ID and price
        QRegularExpression orderIdRegex(R"(#(\d+)\$(\d+\.\d{2}))");
        QRegularExpressionMatch orderIdMatch = orderIdRegex.match(entry);
        if (orderIdMatch.hasMatch()) {
            order.orderId = orderIdMatch.captured(1);
            order.price = orderIdMatch.captured(2);
        }

        // Regular expression to find date ordered
        QRegularExpression dateRegex(R"(Ordered\s+(.*?)(\n|$))");
        QRegularExpressionMatch dateMatch = dateRegex.match(entry);
        if (dateMatch.hasMatch()) {
            order.orderDate = dateMatch.captured(1).trimmed();
        }

        // Regular expression to find item names and quantities
        QRegularExpression itemRegex(R"((.*?)\nQuantity(\d+))");
        QRegularExpressionMatchIterator itemIt = itemRegex.globalMatch(entry);
        while (itemIt.hasNext()) {
            QRegularExpressionMatch itemMatch = itemIt.next();
            QString itemName = itemMatch.captured(1).trimmed();
            int quantity = itemMatch.captured(2).toInt();
            order.itemNames.append(itemName);
            order.quantities.append(quantity);
        }

        // Only add the order if it has valid information
        if (!order.orderId.isEmpty() && !order.quantities.isEmpty()) {
            orders.append(order);
        }
    }

    return orders;
}
-----***** mainwindow.h *****-----
#ifndef MAINWINDOW_H
#define MAINWINDOW_H

#include <QMainWindow>
#include <QChartView>
#include <QScrollArea>
#include <QBarSeries>
#include <QBarSet>
#include <QValueAxis>
#include <QCategoryAxis>

QT_BEGIN_NAMESPACE
namespace Ui {
class MainWindow;
}
QT_END_NAMESPACE

class MainWindow : public QMainWindow
{
    Q_OBJECT

public:
    explicit MainWindow(QWidget *parent = nullptr);
    ~MainWindow();

private:
    Ui::MainWindow *ui;

    QScrollArea *chartScrollArea;  // Scroll area to wrap the chart
    QChartView *chartView;         // Chart view to display the chart

    void setupChart();      // Function to set up the chart
    void loadOrderData();   // Function to load parsed order data

    struct OrderInfo {
        QString name;
        QString orderId;        // Order ID
        QString price;          // Total price of the order
        QString shipDate;       // Shipping date
        QString orderDate;      // Date the order was placed
        QString shippingMethod; // Shipping method
        QString address;        // Shipping address
        QStringList itemNames;  // List of item names
        QList<int> quantities;  // Quantities of each item
    };

    QList<OrderInfo> parseOrderText(const QString &text);  // Order parsing function
};

#endif // MAINWINDOW_H
-----***** mainwindow.ui *****-----
<?xml version="1.0" encoding="UTF-8"?>
<ui version="4.0">
 <class>MainWindow</class>
 <widget class="QMainWindow" name="MainWindow">
  <property name="geometry">
   <rect>
    <x>0</x>
    <y>0</y>
    <width>1100</width>
    <height>736</height>
   </rect>
  </property>
  <property name="windowTitle">
   <string>MainWindow</string>
  </property>
  <widget class="QWidget" name="centralwidget">
   <widget class="QTextEdit" name="orderTextInput">
    <property name="geometry">
     <rect>
      <x>2</x>
      <y>12</y>
      <width>1081</width>
      <height>61</height>
     </rect>
    </property>
   </widget>
   <widget class="QPushButton" name="parseButton">
    <property name="geometry">
     <rect>
      <x>110</x>
      <y>80</y>
      <width>871</width>
      <height>32</height>
     </rect>
    </property>
    <property name="text">
     <string>Process Orders:</string>
    </property>
   </widget>
   <widget class="QWidget" name="graphContainer" native="true">
    <property name="geometry">
     <rect>
      <x>12</x>
      <y>125</y>
      <width>1076</width>
      <height>241</height>
     </rect>
    </property>
   </widget>
   <widget class="QTableWidget" name="orderTable">
    <property name="geometry">
     <rect>
      <x>2</x>
      <y>370</y>
      <width>1081</width>
      <height>291</height>
     </rect>
    </property>
   </widget>
  </widget>
  <widget class="QMenuBar" name="menubar">
   <property name="geometry">
    <rect>
     <x>0</x>
     <y>0</y>
     <width>1100</width>
     <height>38</height>
    </rect>
   </property>
  </widget>
  <widget class="QStatusBar" name="statusbar"/>
 </widget>
 <resources/>
 <connections/>
</ui>
-----***** web2.pro *****-----
QT += core gui widgets charts

CONFIG += c++17

SOURCES += \
    main.cpp \
    mainwindow.cpp

HEADERS += \
    mainwindow.h

FORMS += \
    mainwindow.ui
-----***** END *****-----
