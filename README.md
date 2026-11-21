# zcc
<Window xmlns="https://github.com/avaloniaui"
        xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
        x:Class="WpfApp1.LoginWindow"
        x:CompileBindings="False"
        Title="Авторизация" Width="350" Height="260" WindowStartupLocation="CenterScreen">
    <StackPanel Margin="25" VerticalAlignment="Center" Spacing="10">
        <TextBlock Text="ВХОД В СИСТЕМУ" FontSize="16" FontWeight="Bold" HorizontalAlignment="Center"/>

        <TextBox x:Name="TbLogin" Watermark="Введите логин"/>

        <Button Content="Войти" HorizontalAlignment="Stretch" Background="#0000FF" Foreground="White" Click="BtnLogin_Click"/>
        <Button Content="Войти как гость" HorizontalAlignment="Stretch" Click="BtnGuest_Click"/>

        <TextBlock x:Name="TbError" Foreground="Red" HorizontalAlignment="Center"/>
    </StackPanel>
</Window>

using System.Linq;
using Avalonia.Controls;
using Avalonia.Interactivity;

namespace WpfApp1
{
    public partial class LoginWindow : Window
    {
        public LoginWindow() => InitializeComponent();

        private void BtnLogin_Click(object? sender, RoutedEventArgs e)
        {
            // Вход только по логину
            var u = AppConnect.db.Users.FirstOrDefault(x => x.Login == TbLogin.Text);
            if (u == null)
            {
                TbError.Text = "Пользователь с таким логином не найден!";
                return;
            }

            AppConnect.user = u;
            OpenCatalog();
        }

        private void BtnGuest_Click(object? sender, RoutedEventArgs e)
        {
            AppConnect.user = null;
            OpenCatalog();
        }

        private void OpenCatalog()
        {
            new ProductsWindow().Show();
            Close();
        }
    }
}

карточка прод
<UserControl xmlns="https://github.com/avaloniaui"
             xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
             x:Class="WpfApp1.ProductCard"
             x:CompileBindings="False"
             Margin="4">
    <Border x:Name="CardBorder" BorderBrush="Gray" BorderThickness="1" CornerRadius="4" Padding="10" Background="White">
        <Grid ColumnDefinitions="120, *, 110">
            <!-- 1. Фото слева -->
            <Image x:Name="ImgPhoto" Grid.Column="0" Width="100" Height="100" Stretch="Uniform"/>

            <!-- 2. Информация по центру -->
            <StackPanel Grid.Column="1" VerticalAlignment="Center" Spacing="3">
                <TextBlock x:Name="TbTitle" FontWeight="Bold" FontSize="14"/>
                <TextBlock x:Name="TbCategory"/>
                <TextBlock x:Name="TbCount"/>
                <TextBlock x:Name="TbSostav" TextTrimming="CharacterEllipsis"/>
            </StackPanel>

            <!-- 3. Цена справа -->
            <TextBlock x:Name="TbPrice" Grid.Column="2" FontSize="15" FontWeight="Bold" 
                       VerticalAlignment="Center" HorizontalAlignment="Right"/>
        </Grid>
    </Border>
</UserControl>
using System;
using System.IO;
using System.Linq;
using Avalonia.Controls;
using Avalonia.Media;
using Avalonia.Media.Imaging;
using WpfApp1.Models;

namespace WpfApp1
{
    public partial class ProductCard : UserControl
    {
        public ProductCard()
        {
            InitializeComponent();
            DataContextChanged += (s, e) => UpdateCard();
        }

        private void UpdateCard()
        {
            if (DataContext is not Tovar t) return;

            // 1. Тексты по макету КИМ
            TbTitle.Text = $"{t.ManufacturerNavigation?.Name} | {t.Name}";
            TbCategory.Text = $"Категория: {t.CategoryNavigation?.Name}";
            TbSostav.Text = $"Состав: {t.Description}";

            // 2. Количество: "много" если > 5, "мало" если <= 5
            int count = t.Count ?? 0;
            TbCount.Text = count > 5 ? "Количество: много" : "Количество: мало";

            // 3. Подсветка светло-красным, если остаток <= 3
            CardBorder.Background = count <= 3 ? Brushes.LightCoral : Brushes.White;

            // 4. Расчет цены со скидкой 25%, если не было заказов в прошлом месяце
            var lastMonth = DateTime.Now.AddMonths(-1);
            bool hasOrdersLastMonth = AppConnect.db.OrderItems.Any(oi =>
                oi.TovarId == t.Id &&
                oi.OrderNavigation.OrderDate.Month == lastMonth.Month &&
                oi.OrderNavigation.OrderDate.Year == lastMonth.Year);

            decimal price = t.Price ?? 0;
            if (!hasOrdersLastMonth)
                price = price * 0.75m; // скидка 25%

            TbPrice.Text = $"{price:N2} ₽";

            // 5. Фото или заглушка picture.png
            string path = File.Exists($"Images/{t.Photo}") ? $"Images/{t.Photo}" : "picture.png";
            if (File.Exists(path))
                ImgPhoto.Source = new Bitmap(path);
        }
    }
}

вывод тов

<Window xmlns="https://github.com/avaloniaui"
        xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
        xmlns:local="clr-namespace:WpfApp1"
        x:Class="WpfApp1.ProductsWindow"
        x:CompileBindings="False"
        Title="Каталог обуви" Width="850" Height="600" WindowStartupLocation="CenterScreen">
    <Grid RowDefinitions="45, 45, *, 45">
        <!-- 0. Шапка (ФИО в правом верхнем углу) -->
        <DockPanel Grid.Row="0" Background="#00FFFF" LastChildFill="False">
            <TextBlock Text="КАТАЛОГ ОБУВИ" FontWeight="Bold" FontSize="16" VerticalAlignment="Center" Margin="10,0"/>
            <StackPanel DockPanel.Dock="Right" Orientation="Horizontal" Spacing="10" Margin="10,0" VerticalAlignment="Center">
                <Button x:Name="BtnOrders" Content="Заказы" Click="BtnOrders_Click"/>
                <TextBlock x:Name="TbFio" FontWeight="Bold" VerticalAlignment="Center"/>
                <Button Content="Выйти" Click="BtnLogout_Click"/>
            </StackPanel>
        </DockPanel>

        <!-- 1. Фильтрация, поиск и сортировка -->
        <StackPanel Grid.Row="1" x:Name="FilterPanel" Orientation="Horizontal" Spacing="10" Background="#F0F0F0" VerticalAlignment="Center">
            <TextBox x:Name="TbSearch" Width="180" Watermark="Поиск..." Margin="10,0,0,0" TextChanged="Filter_Changed"/>
            <ComboBox x:Name="CbCategory" Width="160" SelectionChanged="Filter_Changed"/>
            <ComboBox x:Name="CbSort" Width="170" SelectedIndex="0" SelectionChanged="Filter_Changed">
                <ComboBoxItem Content="Без сортировки"/>
                <ComboBoxItem Content="Цена (по возрастанию)"/>
                <ComboBoxItem Content="Цена (по убыванию)"/>
            </ComboBox>
        </StackPanel>

        <!-- 2. Список товаров -->
        <ListBox Grid.Row="2" x:Name="LbProducts" HorizontalAlignment="Stretch">
            <ListBox.ItemTemplate>
                <DataTemplate>
                    <local:ProductCard/>
                </DataTemplate>
            </ListBox.ItemTemplate>
        </ListBox>

        <!-- 3. Кнопка добавления товара -->
        <Button Grid.Row="3" x:Name="BtnAdd" Content="Добавить товар" HorizontalAlignment="Right" 
                Margin="0,0,10,0" Background="#0000FF" Foreground="White" Click="BtnAdd_Click"/>
    </Grid>
</Window>

using System.Collections.Generic;
using System.Linq;
using Avalonia.Controls;
using Avalonia.Interactivity;
using Microsoft.EntityFrameworkCore;
using WpfApp1.Models;

namespace WpfApp1
{
    public partial class ProductsWindow : Window
    {
        private List<Tovar> _all = new();

        public ProductsWindow()
        {
            InitializeComponent();

            // ФИО в правом верхнем углу
            TbFio.Text = AppConnect.user != null ? AppConnect.user.Fio : "Гость";

            // Фильтры доступны авторизованным пользователям
            bool isAuth = AppConnect.user != null;
            FilterPanel.IsVisible = isAuth;
            BtnOrders.IsVisible = isAuth;
            BtnAdd.IsVisible = isAuth;

            // Заполняем категории ("Все категории" первым элементом)
            var cats = new List<string> { "Все категории" };
            cats.AddRange(AppConnect.db.Categories.Select(c => c.Name));
            CbCategory.ItemsSource = cats;
            CbCategory.SelectedIndex = 0;

            LoadData();
        }

        public void LoadData()
        {
            _all = AppConnect.db.Tovars
                .Include(t => t.CategoryNavigation)
                .Include(t => t.ManufacturerNavigation)
                .ToList();

            Filter();
        }

        private void Filter_Changed(object? sender, object e) => Filter();

        private void Filter()
        {
            if (LbProducts == null) return;

            var data = _all.AsEnumerable();

            // 1. Поиск по наименованию и описанию
            string q = TbSearch.Text?.Trim().ToLower() ?? "";
            if (!string.IsNullOrEmpty(q))
            {
                data = data.Where(x =>
                    (x.Name != null && x.Name.ToLower().Contains(q)) ||
                    (x.Description != null && x.Description.ToLower().Contains(q)));
            }

            // 2. Фильтр по категории
            if (CbCategory.SelectedIndex > 0 && CbCategory.SelectedItem is string selectedCat)
            {
                data = data.Where(x => x.CategoryNavigation?.Name == selectedCat);
            }

            // 3. Сортировка по цене
            if (CbSort.SelectedIndex == 1) data = data.OrderBy(x => x.Price);
            if (CbSort.SelectedIndex == 2) data = data.OrderByDescending(x => x.Price);

            LbProducts.ItemsSource = data.ToList();
        }

        private async void BtnAdd_Click(object? sender, RoutedEventArgs e)
        {
            await new AddProductWindow().ShowDialog(this);
            LoadData(); // обновляем список после закрытия окна добавления
        }

        private void BtnOrders_Click(object? sender, RoutedEventArgs e)
        {
            new OrdersWindow().Show();
            Close();
        }

        private void BtnLogout_Click(object? sender, RoutedEventArgs e)
        {
            AppConnect.user = null;
            new LoginWindow().Show();
            Close();
        }
    }
}

доб тов
<Window xmlns="https://github.com/avaloniaui"
        xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
        x:Class="WpfApp1.AddProductWindow"
        x:CompileBindings="False"
        Title="Добавление товара" Width="400" Height="420" WindowStartupLocation="CenterScreen">
    <StackPanel Margin="20" Spacing="8">
        <TextBlock Text="НОВЫЙ ТОВАР" FontWeight="Bold" FontSize="15" HorizontalAlignment="Center"/>

        <TextBox x:Name="TbName" Watermark="Наименование"/>
        <ComboBox x:Name="CbCategory" HorizontalAlignment="Stretch" PlaceholderText="Выберите категорию"/>
        <ComboBox x:Name="CbManufacturer" HorizontalAlignment="Stretch" PlaceholderText="Выберите производителя"/>
        <TextBox x:Name="TbPrice" Watermark="Цена"/>
        <TextBox x:Name="TbCount" Watermark="Количество на складе"/>
        <TextBox x:Name="TbDesc" Watermark="Состав / Описание"/>

        <TextBlock x:Name="TbError" Foreground="Red"/>

        <StackPanel Orientation="Horizontal" HorizontalAlignment="Right" Spacing="10" Margin="0,10,0,0">
            <Button Content="Сохранить" Background="#0000FF" Foreground="White" Click="BtnSave_Click"/>
            <Button Content="Отмена" Click="BtnCancel_Click"/>
        </StackPanel>
    </StackPanel>
</Window>

using System.Linq;
using Avalonia.Controls;
using Avalonia.Interactivity;
using WpfApp1.Models;

namespace WpfApp1
{
    public partial class AddProductWindow : Window
    {
        public AddProductWindow()
        {
            InitializeComponent();

            // Загружаем списки из БД
            CbCategory.ItemsSource = AppConnect.db.Categories.ToList();
            CbCategory.DisplayMemberBinding = new Avalonia.Data.Binding("Name");

            CbManufacturer.ItemsSource = AppConnect.db.Manufacturers.ToList();
            CbManufacturer.DisplayMemberBinding = new Avalonia.Data.Binding("Name");
        }

        private void BtnSave_Click(object? sender, RoutedEventArgs e)
        {
            if (string.IsNullOrWhiteSpace(TbName.Text) ||
                CbCategory.SelectedItem is not Category cat ||
                CbManufacturer.SelectedItem is not Manufacturer manuf ||
                !decimal.TryParse(TbPrice.Text, out decimal price) ||
                !int.TryParse(TbCount.Text, out int count))
            {
                TbError.Text = "Проверьте правильность заполнения всех полей!";
                return;
            }

            var item = new Tovar
            {
                Name = TbName.Text.Trim(),
                CategoryId = cat.Id,
                ManufacturerId = manuf.Id,
                Price = price,
                Count = count,
                Description = TbDesc.Text?.Trim()
            };

            AppConnect.db.Tovars.Add(item);
            AppConnect.db.SaveChanges();
            Close();
        }

        private void BtnCancel_Click(object? sender, RoutedEventArgs e) => Close();
    }
}

закказ

<Window xmlns="https://github.com/avaloniaui"
        xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
        x:Class="WpfApp1.OrdersWindow"
        x:CompileBindings="False"
        Title="Список заказов" Width="700" Height="500" WindowStartupLocation="CenterScreen">
    <Grid RowDefinitions="45, *">
        <StackPanel Grid.Row="0" Orientation="Horizontal" Background="#00FFFF" Spacing="15" VerticalAlignment="Center">
            <TextBlock Text="СПИСОК ЗАКАЗОВ" FontWeight="Bold" FontSize="15" VerticalAlignment="Center" Margin="10,0"/>
            <Button Content="Назад" Click="BtnBack_Click"/>
        </StackPanel>

        <ListBox Grid.Row="1" x:Name="LbOrders" Margin="10">
            <ListBox.ItemTemplate>
                <DataTemplate>
                    <Border BorderBrush="Gray" BorderThickness="1" CornerRadius="4" Padding="8" Margin="0,0,0,5">
                        <StackPanel Spacing="3">
                            <TextBlock Text="{Binding Id, StringFormat='Заказ №{0}'}" FontWeight="Bold"/>
                            <TextBlock Text="{Binding OrderDate, StringFormat='Дата заказа: {0:dd.MM.yyyy}'}"/>
                            <TextBlock Text="{Binding UserNavigation.Fio, StringFormat='Клиент: {0}'}" Foreground="Blue"/>
                        </StackPanel>
                    </Border>
                </DataTemplate>
            </ListBox.ItemTemplate>
        </ListBox>
    </Grid>
</Window>

using System.Linq;
using Avalonia.Controls;
using Avalonia.Interactivity;
using Microsoft.EntityFrameworkCore;

namespace WpfApp1
{
    public partial class OrdersWindow : Window
    {
        public OrdersWindow()
        {
            InitializeComponent();

            // Подгружаем заказы с клиентами
            LbOrders.ItemsSource = AppConnect.db.Orders
                .Include(o => o.UserNavigation)
                .OrderByDescending(o => o.OrderDate)
                .ToList();
        }

        private void BtnBack_Click(object? sender, RoutedEventArgs e)
        {
            new ProductsWindow().Show();
            Close();
        }
    }
}




























