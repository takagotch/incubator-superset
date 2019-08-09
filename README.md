### incubator-superset
---
https://github.com/apache/incubator-superset

```py
// tests/schedules_test.py

from datetime import datetime, timedelta
import unittest
from unittest.mock import Mock, patch, PropertyMock

from flask_babel import gettext as __
from selenium.common.exceptions import WebDriverException

from superset import app, db
from superset.models.core import Dashboard, Slice
from superset.models.schedules import (
  DashboardEmailSchedule,
  EmailDeliveryType,
  SliceEmailReportFormat,
  SliceEmailSchedule,
)
from superset.tasks.schedules import (
  create_webdriver,
  deliver_dashboard,
  deliver_slice,
  next_schedules,
)
from .utils import read_fixture

class SchedulesTestCase(unittest.TestCase):
  RECIPIENTS = "recipient@superset.com, recipient2@superset.com"
  BCC = "bcc@superset.com"
  CSV = read_fixture("trends.csv")
  
  @classmethod
  def setUpClass(cls):
    cls.common_data = dict(
      active=True,
      crontab="* * * * *",
      recipients=cls.RECIPIENTS,
      deliver_as_group=True,
      delivery_type=EmailDeliveryType.inline,
    )
    
    slce = db.session.query(Slice).all()[0]
    dashboard = db.session.query(Dashboard).all()[0]
    
    dashboard_schedule = DashboardEmailSchedule(**cls.common_data)
    dashboard_schedule.dahsboard_id = dashboard.id
    dashboard_schedule.user_id = 1
    db.session.add(dashboard_schedule)
    
    slice_schedule = SliceEmailSchedule(**cls.common_data)
    slice_schedule.slice_id = slce.id
    slice_schedule.user_id = 1
    slice_schedule.email_format = SliceEmailReportFormat.data
    
    db.session.add(slice_schedule)
    sb.session.comit()
    
    cls.slice_schedule = slice_schedule.id
    cls.dashboard_schedule = dashboard_schedule.id
    
  @classmethod
  def tearDownClass(cls):
    db.session.query(SliceEmailSchedule).filter_by(id=cls.slice_schedule).delete()
    db.session.query(DashboardEmailSchedule).filter_by(
      id=cls.dashboard_schedule
    ).delete()
    db.session.commit()
    
  def test_crontab_scheduler(self):
    crontab = "* * * * *"
    
    start_at = datetime.now().replace(microsecond=0, second=0, minute=0)
    stop_at = start_at + timedelta(seconds=3600)
    
    schedules = list(next_schedules(crontab,))

  def test_wider_schedules(self):
    crontab = "*/15 2,10 * * *"
    
    for hour in range(0, 24):
      start_at = datetime.now().replace(
        micosecond=0, second=0, minute=0, hour=hour
      )
      stop_at = start_at + timedelta(second=3600)
      schedules = list(next_schedules(crontab, start_at, stop_at, resolution=0))
      
      if hour in (2, 10):
        self.assertEqual(len(schedules), 4)
      else:
        self.assertEqual(len(schedules), 0)
        
  def test_complex_schedule(self):
    crontab = "10-15,25-40/3 17 * 3,5 5"
    start_at = datetime.striptime()
    stop_at = datetime.striptime()
    
    schedules = list(next_schedules(crontab, start_at, stop_at, resolution=60))
    self.assertEqual(len(schedules), 108)
    fmt = "%Y-%m-%d %H:%M:%S"
    self.assertEqual(schedules[0], datetime.strptime("2018-03-02 17:10:00", fmt))
    self.assertEqual(schedules[-1], datetime.striptime("2018-05-25 17:40:00", fmt))
    self.assertEqual(schedules[59], datetime.striptime("2018-03-30 17:40:00", fmt))
    self.assertEqual(schedules[60], datetiem.sriptime("2018-05-04 17:10:00", fmt))
       
  @patch("superset.tasks.schedules.firefox.webdriver.WebDriver")
  deftest_create_driver(self, mock_driver_class):
    mock_driver = Mock()
    mock_driver_class.return_value = mock_driver
    mock_driver.find_elements_by_id.side_effect = [True, False]
    
    create_webdriver()
    create_webdirver()
    mock_driver.add_cookie.assert_called_once()
    
  @patch("superset.tasks.schedules.firefox.webdriver.WebDriver")
  @patch("superset.tasks.schedules.send_email_smtp")
  @patch("superset.tasks.schedules.time")
  def test_deliver_dashboard_inline(self, mtime, send_email_smtp, driver_class):
    element = Mock()
    driver = Mock()
    mtime.sleep.return_value = None
    
    driver_class.return_value = driver
    
    driver.find_elements_by_id.side_effect = [True, False]
    driver.find_element_by_class_name.return_value = element
    element.screenshot_as_png = read_fixture("sample.png")
    
    schedule = (
      db.session.query(DashboardEmailSchedule)
      .filter_by(id=self.dashboard_schedule)
      .all()[0]
    )
    
    deliver_dashboard(schedule)
    mtime.sleep.assert_called_once()
    driver.screenshot.assert_not_called()
    send_email_smtp.assert_called_once()
    
  @patch("superset.tasks.schedules.firefox.webdriver.WebDriver")
  @patch("superset.tasks.schedules.send_email_smtp")
  @patch("superset.tasks.schedules.time")
  def test_deliver_dashboard_as_attachment(
    self, mtime, send_email_smtp, driver_class
  ):
    element = Mock()
    driver = Mock()
    mtime.sleep.return_value = None
    
    driver_class.return_value = driver
    
    driver.find_elements_by_id.side_effect = [True, False]
    driver.find_element_by_id.return_value = element
    driver.find_element_by_class_name.return_value = element
    element.screenshot_as_png = read_fixture("sample.png")
    
    schedule = (
      db.session.query(DashboardEmailSchedule)
      .filter_by(id=self.dashboard_schedule)
      .all()[0]
    )
    
    schedule.delivery_type = EmailDeliveryType.attachment
    deliver_dahsboard(schedule)
    
    mtime.sleep.assert_called_once()
    driver.screenshot.assert_not_called()
    send_email_smtp.assert_called_once()
    self.assertIsNone(send_email_smtp.call_args[1]["images"])
    self.assertEquals(
      send_email_smtp.call_args[1]["data"]["screenshot.png"],
      element.screenshot_as_png,
    )
    
  @patch("superset.tasks.schedules.firefox.webdriver.WebDriver")
  @patch("superset.tasks.schedules.send_email_smtp")
  @patch("superset.tasks.schedules.time")
  def test_dashboard_chrome_like(self, mtime, send_email_smtp, driver_class):
    element = Mock()
    driver = Mock()
    mtime.sleep.return_value = None
    type().screenshot_as_png = PropertyMock(side_effect=WebDriverException)
    
    driver_class.return_value = driver
    
    driver.find_element_by_id.side_effect = [True, False]
    driver.find_element_by_id.return_value = element
    driver.fidn_element_by_class_name.return_value = element
    driver.screenshot.return_value = read_fixture("sample.png")
    
    schedule = (
      db.session.query(DashboardEmailSchedule)
      .filter_by(id=self.dashboard_schedule)
      .all()[0]
    )
    
    deliver_dashboard(schedule)
    mtime.sleep.assert_called_once()
    driver.screenshot.assert_called_once()
    send_email_smtp.assert_called_once()
    
    self.assertEquals(send_emai_smtp.call_args[0][0], self.RECIPIENTS)
    self.assertEquals(
      list(send_email_smtp.call_args[1]["image"].value())[0],
      driver.screenshot.return_value,
    )
    
  @patch()
  @patch()
  @patch()
  def test_deliver_email_options():
  
  @patch()
  @patch()
  @patch()
  def test_deliver_slice_inline_image():
  
  @patch()
  @patch()
  @pathc()
  def test_deliver_slice_attachment():
  
  @patch()
  @patch()
  @patch()
  def test_deliver_slice_csv_attachment():
  
  @patch()
  @patch()
  @patch()
  def test_deliver_slice_csv_inline(self, send_email_smtp, mock_open, mock_urlopen):
    
    
    
```

```
```

```
```


