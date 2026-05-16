# shop-management-app
Bengali Accounting App for Small Shops
import streamlit as st
import pandas as pd
import datetime
import os

# ইউজার ডেটা থাকলে লোড করুন, না থাকলে তৈরি করুন
if not os.path.exists('users.csv'):
    pd.DataFrame({
        'shop_name': ['দোকান১', 'দোকান২'],
        'password': ['pass1', 'pass2']
    }).to_csv('users.csv', index=False)

users_df = pd.read_csv('users.csv')

# লগইন স্টেট সেটআপ
if 'logged_in' not in st.session_state:
    st.session_state['logged_in'] = False
    st.session_state['shop_name'] = ''

# লগইন ফর্ম
if not st.session_state['logged_in']:
    st.title("লগইন করুন")
    with st.form("login_form"):
        shop = st.text_input("দোকানের নাম")
        pwd = st.text_input("পাসওয়ার্ড", type='password')
        login_button = st.form_submit_button("লগইন")
        if login_button:
            if shop in users_df['shop_name'].values:
                correct_pwd = users_df[users_df['shop_name'] == shop]['password'].values[0]
                if pwd == correct_pwd:
                    st.session_state['logged_in'] = True
                    st.session_state['shop_name'] = shop
                else:
                    st.error("ভুল পাসওয়ার্ড")
            else:
                st.error("অজানা দোকানের নাম")
else:
    # লগআউট বোতাম
    if st.button("লগআউট"):
        st.session_state['logged_in'] = False
        st.session_state['shop_name'] = ''
        st.experimental_rerun()

    shop_name = st.session_state['shop_name']
    st.success(f"স্বাগতম, {shop_name}")

    # ডেটা লোড করার ফাংশন
    def load_shop_data(file_name, columns):
        if os.path.exists(file_name):
            return pd.read_csv(file_name)
        else:
            return pd.DataFrame(columns=columns)

    # ডেটা ফাইলের নাম নির্ধারণ
    sales_file = f"{shop_name}_sales.csv"
    supplier_file = f"{shop_name}_supplier.csv"
    customer_file = f"{shop_name}_customer.csv"

    # ডেটা লোড করুন
    sales_df = load_shop_data(sales_file, ['তারিখ', 'বিবরণ', 'পেমেন্ট মেথড', 'ইন (টাকা)', 'আউট (টাকা)'])
    supplier_df = load_shop_data(supplier_file, ['তারিখ', 'মহাজনের নাম', 'মোট বিল', 'পরিশোধিত', 'পাওনা'])
    customer_df = load_shop_data(customer_file, ['তারিখ', 'কাস্টমারের নাম', 'মালের দাম', 'জমা দিয়েছে', 'বাকি'])

    # --- সাইডবার মেনু ---
    menu = [
        "📊 লাইভ ড্যাশবোর্ড",
        "💰 দৈনন্দিন লেনদেন",
        "🤝 মহাজন/পার্টি হিসাব",
        "👤 কাস্টমার বাকি হিসাব",
        "🏦 ওয়ালেট ও ব্যাংক ব্যালেন্স"
    ]
    choice = st.sidebar.selectbox("মেনু সিলেক্ট করুন", menu)

    # --- ১. লাইভ ড্যাশবোর্ড ---
    if choice == "📊 লাইভ ড্যাশবোর্ড":
        st.subheader("দোকানের সার্বিক অবস্থা")
        total_cash_in = sales_df['ইন (টাকা)'].sum()
        total_cash_out = sales_df['আউট (টাকা)'].sum()
        net_cash = total_cash_in - total_cash_out
        total_supplier_due = supplier_df['পাওনা'].sum()
        total_customer_due = customer_df['বাকি'].sum()

        col1, col2, col3 = st.columns(3)
        col1.metric("মোট ক্যাশ ইন", f"{total_cash_in} টাকা")
        col2.metric("মোট ক্যাশ আউট", f"{total_cash_out} টাকা")
        col3.metric("বর্তমানে হাতে আছে", f"{net_cash} টাকা")

        st.markdown("---")
        c1, c2 = st.columns(2)
        c1.warning(f"🔴 মহাজন আপনার কাছে পাবে: {total_supplier_due} টাকা")
        c2.success(f"🟢 কাস্টমারের কাছে আপনি পাবেন: {total_customer_due} টাকা")

    # --- ২. দৈনন্দিন লেনদেন ---
    elif choice == "💰 দৈনন্দিন লেনদেন":
        st.subheader("নতুন ক্যাশ ইন/আউট এন্ট্রি")
        with st.form("cash_form"):
            date = st.date_input("তারিখ", datetime.date.today())
            desc = st.text_input("বিবরণ (উদা: ৫টি শার্ট বিক্রি)")
            method = st.selectbox("মাধ্যম", ["ক্যাশ", "ব্যাংক", "বিকাশ"])
            t_type = st.radio("ধরণ", ["টাকা আসছে (In)", "টাকা গেছে (Out)"])
            amount = st.number_input("টাকার পরিমাণ", min_value=0)
            if st.form_submit_button("লেনদেন সেভ করুন"):
                v_in = amount if t_type == "টাকা আসছে (In)" else 0
                v_out = amount if t_type == "টাকা গেছে (Out)" else 0
                new_row = {
                    'তারিখ': date,
                    'বিবরণ': desc,
                    'পেমেন্ট মেথড': method,
                    'ইন (টাকা)': v_in,
                    'আউট (টাকা)': v_out
                }
                sales_df = pd.concat([sales_df, pd.DataFrame([new_row])], ignore_index=True)
                sales_df.to_csv(sales_file, index=False)
                st.success("হিসাব আপডেট হয়েছে!")

    # --- ৩. মহাজন/পার্টি হিসাব ---
    elif choice == "🤝 মহাজন/পার্টি হিসাব":
        st.subheader("যাদের থেকে মাল কিনেন (মহাজন)")
        with st.form("sup_form"):
            s_date = st.date_input("তারিখ", datetime.date.today())
            s_name = st.text_input("মহাজনের নাম")
            s_bill = st.number_input("মোট বিল", min_value=0)
            s_paid = st.number_input("পরিশোধ করেছেন", min_value=0)
            if st.form_submit_button("সেভ করুন"):
                s_due = s_bill - s_paid
                new_s = {
                    'তারিখ': s_date,
                    'মহাজনের নাম': s_name,
                    'মোট বিল': s_bill,
                    'পরিশোধিত': s_paid,
                    'পাওনা': s_due
                }
                supplier_df = pd.concat([supplier_df, pd.DataFrame([new_s])], ignore_index=True)
                supplier_df.to_csv(supplier_file, index=False)
                st.balloons()
        st.dataframe(supplier_df)

    # --- ৪. কাস্টমার বাকি হিসাব ---
    elif choice == "👤 কাস্টমার বাকি হিসাব":
        st.subheader("কাস্টমারের কাছে বাকি (পাওনা)")
        with st.form("cust_form"):
            c_date = st.date_input("তারিখ", datetime.date.today())
            c_name = st.text_input("কাস্টমারের নাম/মোবাইল")
            c_price = st.number_input("মালের মোট দাম", min_value=0)
            c_cash = st.number_input("নগদ দিয়েছে", min_value=0)
            if st.form_submit_button("বাকির খাতায় লিখুন"):
                c_due = c_price - c_cash
                new_c = {
                    'তারিখ': c_date,
                    'কাস্টমারের নাম': c_name,
                    'মালের দাম': c_price,
                    'জমা দিয়েছে': c_cash,
                    'বাকি': c_due
                }
                customer_df = pd.concat([customer_df, pd.DataFrame([new_c])], ignore_index=True)
                customer_df.to_csv(customer_file, index=False)
                st.success("কাস্টমারের বাকির হিসাব সেভ হয়েছে!")
        st.dataframe(customer_df)

    # --- ৫. ওয়ালেট ও ব্যাংক ব্যালেন্স ---
    elif choice == "🏦 ওয়ালেট ও ব্যাংক ব্যালেন্স":
        st.subheader("কোথায় কত টাকা আছে")
        for m in ["ক্যাশ", "ব্যাংক", "বিকাশ"]:
            m_in = sales_df[sales_df['পেমেন্ট মেথড'] == m]['ইন (টাকা)'].sum()
            m_out = sales_df[sales_df['পেমেন্ট মেথড'] == m]['আউট (টাকা)'].sum()
            balance = m_in - m_out
            st.info(f"💳 {m}-এ বর্তমান ব্যালেন্স: {balance} টাকা")
